# Plano: migrar a lib de cache do `cache-manager` do Nest para o `lru-cache`

> Este arquivo é um prompt de implementação. Leia o código atual da lib antes de alterar qualquer coisa e adapte os exemplos abaixo à estrutura real dela. Os trechos de código são referência, não substituem o que já existe.

## 1. Contexto

A lib é um conjunto de decorators de cache no estilo do Spring Cache, feita para APIs NestJS. Hoje ela usa o `cache-manager` do Nest como store. O `cache-manager` exige muito trabalho manual e **não controla o uso de memória**: o cache cresce até o processo estourar.

Estado atual da lib:

- Existe o decorator `@Cacheable`, que recebe o nome do cache, a chave e o TTL por método. Pode precisar de adaptação.
- Existe um **serviço exposto para evict** por chave, que usa o `cache-manager`.
- **Não existem** os decorators `@CachePut` e `@CacheClear` (equivalente ao `@CacheEvict` do Spring).
- Não existe limite de memória.

## 2. Objetivo

1. Trocar o `cache-manager` pelo `lru-cache` como único store, sem depender dele.
2. Impor um **teto de memória em megabytes**, configurável, com despejo automático dos itens menos recentemente usados.
3. Permitir **configuração por grupo (cache name)** via `.env`: TTL, teto em MB e máximo de itens.
4. Manter `@Cacheable` funcionando, adaptando-o ao novo store.
5. Criar `@CachePut` e `@CacheClear`.
6. Adaptar o serviço de evict existente para operar sobre o `lru-cache`.

## 3. Fora de escopo (por enquanto)

- Redis ou qualquer store distribuído.
- Evict por padrão/regex (só se surgir necessidade; ver seção 9).
- Proteção contra cache stampede e cache de resultado nulo (ver seção 9, opcionais).

## 4. Decisões de arquitetura

### 4.1 Uma instância de `LRUCache` por cache name

Cada cache name (ex.: `viacep`, `usuarios`) tem sua própria instância, com teto próprio. Assim um grupo nunca consome a memória de outro.

- Criação **preguiçosa**: a instância nasce na primeira vez que o grupo é usado.
- Um registry em módulo TypeScript puro (`Map<string, LRUCache>`) guarda as instâncias. Não precisa de registro no `app.module.ts`, porque o decorator importa o registry direto.
- Consequência: a soma dos tetos de todos os grupos é o consumo máximo real do cache, fora o overhead do heap.
- Se a lib hoje expõe um `Module`/`Provider`, manter essa fachada e delegar internamente ao registry, para não quebrar quem já usa.

### 4.2 Medição de tamanho

Node não permite medir o tamanho real de um objeto. Usar uma estimativa:

```ts
sizeCalculation: (value) => Buffer.byteLength(JSON.stringify(value) ?? '', 'utf8')
```

- O tamanho real no heap costuma ser maior que o JSON. Documentar que o teto em MB é **aproximado** e deixar folga.
- Permitir uma função `sizeOf` customizada por grupo, opcional, para quem quiser medir diferente.
- Item maior que o `maxSize` do grupo: o `lru-cache` não guarda e não esvazia o cache. Garantir isso com teste.

### 4.3 Política de despejo

O `lru-cache` já despeja o menos recentemente usado quando estoura `max` (itens) ou `maxSize` (bytes). Vale o que estourar primeiro. Não reimplementar nada disso.

### 4.4 Expiração

- TTL padrão por instância e TTL por item no `set`.
- `ttlAutopurge` desligado por padrão (custa um timer por item). Deixar configurável.
- `updateAgeOnGet` opcional, para comportamento de TTI (tempo de ociosidade).
- Itens expirados que ainda não foram removidos não são devolvidos pelo `get`.

## 5. Configuração via `.env`

Ordem de prioridade para cada valor: **variável do grupo → padrão global → valor fixo no código**. O TTL passado direto no decorator vence todos.

```env
# padrões globais
CACHE_DEFAULT_MAX_SIZE_MB=10
CACHE_DEFAULT_MAX_ITEMS=1000
CACHE_DEFAULT_TTL_MS=300000

# por grupo: CACHE_<NOME_DO_CACHE>_<CAMPO>
CACHE_VIACEP_MAX_SIZE_MB=5
CACHE_VIACEP_MAX_ITEMS=2000
CACHE_VIACEP_TTL_MS=1800000

CACHE_USUARIOS_MAX_SIZE_MB=20
CACHE_USUARIOS_TTL_MS=600000
```

Regras:

- O nome do grupo vira prefixo em maiúsculas, trocando qualquer caractere fora de `A-Z0-9` por `_` (ex.: `via-cep` → `CACHE_VIA_CEP_TTL_MS`).
- Valor ausente, vazio ou não numérico é ignorado e cai para o próximo nível.
- Opções globais opcionais: `CACHE_TTL_AUTOPURGE` (`true`/`false`) e `CACHE_UPDATE_AGE_ON_GET` (`true`/`false`).
- **Atenção à unidade do TTL.** Verifique em que unidade o `@Cacheable` e o `cache-manager` atuais recebem o TTL (segundos ou milissegundos, depende da versão). O `lru-cache` trabalha em **milissegundos**. Preserve a unidade da API pública da lib e converta internamente, ou então documente a quebra de compatibilidade.
- As variáveis precisam estar carregadas antes da primeira chamada ao registry. Como a criação é preguiçosa, basta o `ConfigModule.forRoot()` ou `import 'dotenv/config'` rodar na inicialização.

## 6. Implementação de referência

### 6.1 Config por grupo

```ts
// cache.config.ts
export interface GroupConfig {
  maxSizeMb: number;
  maxItems: number;
  ttlMs: number;
}

function readNumber(key: string): number | undefined {
  const raw = process.env[key];
  if (raw === undefined || raw === '') return undefined;
  const n = Number(raw);
  return Number.isNaN(n) ? undefined : n;
}

export function resolveGroupConfig(cacheName: string): GroupConfig {
  const prefix = `CACHE_${cacheName.toUpperCase().replace(/[^A-Z0-9]/g, '_')}`;
  return {
    maxSizeMb:
      readNumber(`${prefix}_MAX_SIZE_MB`) ?? readNumber('CACHE_DEFAULT_MAX_SIZE_MB') ?? 10,
    maxItems:
      readNumber(`${prefix}_MAX_ITEMS`) ?? readNumber('CACHE_DEFAULT_MAX_ITEMS') ?? 1000,
    ttlMs:
      readNumber(`${prefix}_TTL_MS`) ?? readNumber('CACHE_DEFAULT_TTL_MS') ?? 300_000,
  };
}
```

### 6.2 Registry

```ts
// cache.registry.ts
import { LRUCache } from 'lru-cache';
import { resolveGroupConfig } from './cache.config';

const stores = new Map<string, LRUCache<string, any>>();

export function getStore(cacheName: string): LRUCache<string, any> {
  const existente = stores.get(cacheName);
  if (existente) return existente;

  const cfg = resolveGroupConfig(cacheName);
  const store = new LRUCache<string, any>({
    max: cfg.maxItems,
    maxSize: cfg.maxSizeMb * 1024 * 1024,
    sizeCalculation: (value) => Buffer.byteLength(JSON.stringify(value) ?? '', 'utf8'),
    ttl: cfg.ttlMs,
    ttlAutopurge: process.env.CACHE_TTL_AUTOPURGE === 'true',
    updateAgeOnGet: process.env.CACHE_UPDATE_AGE_ON_GET === 'true',
    dispose: (_value, key, reason) => {
      // reason: 'evict' | 'set' | 'delete' | 'expire' | 'fetch'
      // 'set' dispara em sobrescrita; filtre no log para não gerar ruído
      if (reason !== 'set') {
        console.log(`[cache:${cacheName}] removido ${key} motivo=${reason}`);
      }
    },
  });

  stores.set(cacheName, store);
  return store;
}

export function statsPorGrupo() {
  return Array.from(stores.entries()).map(([nome, store]) => ({
    cacheName: nome,
    itens: store.size,
    bytesUsados: store.calculatedSize,
    bytesMaximo: store.maxSize,
  }));
}

export function limparGrupo(cacheName: string) {
  stores.get(cacheName)?.clear();
}
```

Trocar o `console.log` pelo logger que a lib já usa (por exemplo o `Logger` do Nest).

### 6.3 `@Cacheable`

Adaptar o decorator existente mantendo sua assinatura pública (nome do cache, chave, TTL). Comportamento esperado:

- Monta a chave com a função `key` recebida ou, na ausência dela, `args.join(':')`.
- Consulta o store do grupo. Se houver valor, devolve sem executar o método.
- Se não houver, executa o método original, guarda o resultado e devolve.
- O TTL do decorator, quando informado, é passado no `set` como `{ ttl }` e sobrescreve o do grupo. Sem ele, vale o TTL do grupo.
- Não cacheia `undefined`. Sobre `null`, ver seção 9.

```ts
descriptor.value = async function (...args: any[]) {
  const store = getStore(options.cacheName);
  const key = options.key ? options.key(...args) : args.join(':');

  const cached = store.get(key);
  if (cached !== undefined) return cached;

  const result = await original.apply(this, args);
  if (result !== undefined && result !== null) {
    store.set(key, result, options.ttlMs !== undefined ? { ttl: options.ttlMs } : undefined);
  }
  return result;
};
```

### 6.4 `@CachePut` (novo)

Semântica do Spring: **sempre executa o método** e grava o resultado no cache, atualizando a entrada. Mesma assinatura de opções do `@Cacheable` (nome do cache, chave, TTL opcional). Como a chave pode depender do resultado ou dos argumentos, definir com cuidado e documentar. Sugestão: a chave é calculada a partir dos argumentos, como no `@Cacheable`.

### 6.5 `@CacheClear` (novo, equivalente ao `@CacheEvict`)

Opções:

- `cacheName`: grupo alvo.
- `key`: função que gera a chave a partir dos argumentos. Remove só essa entrada.
- `allEntries: true`: limpa o grupo inteiro (`store.clear()`).
- `beforeInvocation: true`: remove antes de executar o método. Por padrão, remove **depois** da execução bem-sucedida, para não apagar o cache se o método lançar erro.

### 6.6 Serviço de evict existente

Reescrever sobre o registry, preservando a assinatura pública atual:

- Evict por chave: `getStore(cacheName).delete(key)`.
- Limpar grupo: `getStore(cacheName).clear()`.
- Opcional: evict por prefixo, iterando `store.keys()` e apagando as chaves que começam com o prefixo.

O `@CacheClear` deve reutilizar esse serviço, e não duplicar a lógica.

## 7. Passos de execução

1. Ler a lib inteira: decorators, serviço de evict, módulo, testes, README e qualquer ponto que importe o `cache-manager`.
2. Instalar `lru-cache` (versão atual, exporta `{ LRUCache }`). Conferir a versão instalada e a assinatura de `dispose` e `set` nessa versão.
3. Criar `cache.config.ts` e `cache.registry.ts`.
4. Adaptar o `@Cacheable` para usar o registry. Preservar a assinatura e a unidade do TTL.
5. Reescrever o serviço de evict sobre o registry.
6. Criar `@CachePut` e `@CacheClear`.
7. Remover a dependência do `cache-manager` (código, módulo, `package.json`) se nada mais usar. Se algum consumidor ainda depender dele, sinalizar antes de remover.
8. Atualizar README e `.env.example` com as novas variáveis.
9. Escrever e rodar os testes da seção 8.

## 8. Critérios de aceite e testes

- `@Cacheable`: segunda chamada com os mesmos argumentos não executa o método (HIT); argumentos diferentes geram chaves diferentes.
- TTL: valor expira no tempo configurado. Precedência correta entre decorator, variável do grupo e padrão global.
- Teto de memória: ao ultrapassar `maxSize`, os itens menos recentemente usados saem primeiro e o total em bytes nunca passa do limite.
- Item maior que o limite do grupo não é guardado e não esvazia o cache.
- Isolamento: encher o grupo A não remove nada do grupo B.
- `@CachePut`: o método sempre executa e o valor novo substitui o antigo.
- `@CacheClear`: remoção por chave, `allEntries` e `beforeInvocation`. Se o método lançar erro, o comportamento padrão **não** limpa o cache.
- Serviço de evict: remove por chave e limpa o grupo.
- Variável de ambiente inválida ou vazia cai para o próximo nível de prioridade.
- Nenhum import de `cache-manager` sobra no código.
- `statsPorGrupo()` retorna itens, bytes usados e bytes máximos por grupo.

## 9. Melhorias opcionais (só se fizerem falta)

- **Cache stampede**: guardar num `Map` as promises em andamento por chave, para que chamadas simultâneas ao mesmo valor esperem a mesma promise. Alternativa: usar o `fetchMethod` do `lru-cache`, que já deduplica.
- **Cachear resultado nulo** (ex.: CEP não encontrado) com um marcador e TTL curto, para não martelar a API externa. Hoje `null` não é cacheado.
- **Endpoint de diagnóstico** expondo `statsPorGrupo()`.
- **Evict por prefixo ou padrão**, se o evict por chave exata não bastar.

## 10. Riscos e cuidados

- O limite em MB é estimado por JSON, não é o tamanho real no heap. Deixar folga.
- Valores que não serializam em JSON (referências circulares, `BigInt`) fazem o `JSON.stringify` lançar erro. Tratar no `sizeCalculation` ou documentar a restrição.
- O cache é por processo. Com várias réplicas (por exemplo, pods no Kubernetes), cada uma tem o seu cache, e um evict só afeta a réplica que o recebeu. Documentar isso.
- Mudança de comportamento em relação ao `cache-manager`: confirmar unidade do TTL, serialização dos valores e retorno de `get` para chave inexistente (`undefined` vs `null`).
