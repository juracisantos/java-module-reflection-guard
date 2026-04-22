---
name: java-module-reflection-guard
description: >
  Use esta skill SEMPRE que o usuário mencionar migração de versão do Java (ex: Java 8 para 11, 11 para 17, 17 para 21),
  atualização de dependências Java, erros de reflexão/módulos no Java moderno, problemas com Gson/Jackson/serialização em Java 17+,
  erros como "InaccessibleObjectException", "Failed making field accessible", "module java.base does not open",
  ou qualquer menção a "strong encapsulation", "module system", "java.io.File serialization".
  Também acione quando o usuário perguntar como evitar erros de runtime após migração Java ou pedir auditoria de código
  antes de migrar versões.
---

# Java Migration Compatibility Skill

Skill para detectar, prevenir e corrigir problemas de compatibilidade ao migrar entre versões do Java, com foco especial em **Java 9+ Module System (JPMS)** e **strong encapsulation** que afeta serialização, reflexão e bibliotecas de terceiros.

---

## Por que este skill existe

A partir do **Java 9**, o sistema de módulos (JPMS) foi introduzido. A partir do **Java 17**, o encapsulamento forte foi aplicado com avisos. A partir do **Java 21**, o runtime **bloqueia ativamente** acesso via reflexão a campos internos de classes do JDK (como `java.io.File#path`, `sun.*`, etc.).

Isso causa erros silenciosos em produção que não aparecem em desenvolvimento quando:
- A lista/campo estava `null` em dev mas instanciada em prod
- O ambiente de dev ainda usava flags `--add-opens` herdadas
- A versão anterior do Java permita o acesso com apenas um warning

---

## Categorias de Problemas

### 1. Serialização de tipos JDK com Gson/Jackson
**Sintoma**: `Failed making field 'java.io.File#path' accessible`  
**Causa**: Gson/Jackson usam reflexão para inspecionar campos de objetos. Tipos do JDK como `java.io.File`, `java.nio.Path`, `java.time.*`, `java.net.URL` têm campos internos que o module system bloqueia.

**Tipos JDK problemáticos para serialização direta:**
- `java.io.File`
- `java.nio.file.Path` / `java.nio.file.Paths`
- `java.net.URL`, `java.net.URI`
- `java.time.LocalDate`, `java.time.LocalDateTime`, `java.time.ZonedDateTime`
- `java.util.Currency`
- Qualquer classe de `sun.*` ou `com.sun.*`

### 2. Reflexão em campos privados de classes JDK
**Sintoma**: `InaccessibleObjectException: Unable to make field private ... accessible`  
**Causa**: `Field.setAccessible(true)` em campos de módulos não exportados.

### 3. Bibliotecas de terceiros com reflexão legada
**Bibliotecas comumente afetadas:**
- **Gson** < 2.10: sem adaptadores nativos para tipos JDK
- **Kryo**: serialização binária com reflexão agressiva
- **XStream**: serialização XML via reflexão
- **Hibernate** (versões antigas): proxies e lazy loading
- **Mockito** < 4.x: mocking via reflexão

---

## Checklist de Auditoria Antes da Migração

Antes de migrar, audite o código seguindo estes passos:

### Passo 1 — Buscar tipos JDK em objetos serializados
```bash
# Buscar campos File, Path, URL em classes que são serializadas/enviadas como JSON
grep -rn "java\.io\.File\|java\.nio\.file\.Path\|java\.net\.URL\|java\.net\.URI" src/ \
  --include="*.java" | grep -v "import"
```

### Passo 2 — Identificar uso de Gson/Jackson sem configuração de módulos
```bash
# Verificar se há instâncias de Gson sem TypeAdapters customizados
grep -rn "new Gson()\|GsonBuilder()" src/ --include="*.java"

# Verificar se ObjectMapper tem módulo JavaTimeModule registrado
grep -rn "ObjectMapper\|registerModule\|JavaTimeModule" src/ --include="*.java"
```

### Passo 3 — Verificar flags JVM herdadas
```bash
# Verificar se o projeto depende de --add-opens para funcionar
grep -rn "\-\-add-opens\|--illegal-access" . \
  --include="*.xml" --include="*.gradle" --include="*.sh" --include="*.yml"
```

### Passo 4 — Compilar com `--release` para detectar APIs removidas
```bash
# Maven
mvn compile -Dmaven.compiler.release=21

# Gradle
./gradlew compileJava -Pjava.version=21
```

---

## Correções por Tipo de Problema

### Problema: `List<File>` ou `File` em objeto serializado com Gson

**❌ Código problemático:**
```java
// GsonConfig.java
@Bean
public Gson gson() {
    return new Gson(); // sem customização — falha no Java 17+
}

// Classe com campo File
public class EnvioEmail {
    private List<File> anexos; // java.io.File não é serializável via Gson no Java 21
}

// Uso
gson.toJson(envioEmail); // InaccessibleObjectException em runtime
```

**✅ Solução 1 — TypeAdapter customizado para File (mínima mudança):**
```java
@Bean
public Gson gson() {
    return new GsonBuilder()
        .registerTypeAdapter(File.class, new JsonSerializer<File>() {
            @Override
            public JsonElement serialize(File src, Type typeOfSrc, JsonSerializationContext ctx) {
                return src == null ? JsonNull.INSTANCE : new JsonPrimitive(src.getAbsolutePath());
            }
        })
        .registerTypeAdapter(File.class, new JsonDeserializer<File>() {
            @Override
            public File deserialize(JsonElement json, Type type, JsonDeserializationContext ctx) {
                return json.isJsonNull() ? null : new File(json.getAsString());
            }
        })
        .create();
}
```

**✅ Solução 2 — Substituir `File` por `String` no DTO (recomendada):**
```java
// Antes
public class EnvioEmail {
    private List<File> anexos;
}

// Depois — use String com o path, ou byte[] com o conteúdo
public class EnvioEmail {
    private List<String> caminhoAnexos; // serializa sem problemas
    // ou
    private List<AnexoDTO> anexos; // DTO próprio com campos primitivos
}
```

**✅ Solução 3 — Excluir o campo da serialização:**
```java
public class EnvioEmail {
    @com.google.gson.annotations.Expose(serialize = false, deserialize = false)
    private List<File> anexos;
    // Use GsonBuilder().excludeFieldsWithoutExposeAnnotation()
}
```

---

### Problema: `java.time.*` não serializando corretamente

**✅ Jackson — registrar JavaTimeModule:**
```java
ObjectMapper mapper = new ObjectMapper();
mapper.registerModule(new JavaTimeModule());
mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
```

**✅ Gson — TypeAdapter para LocalDate:**
```java
new GsonBuilder()
    .registerTypeAdapter(LocalDate.class, (JsonSerializer<LocalDate>)
        (src, type, ctx) -> new JsonPrimitive(src.toString()))
    .registerTypeAdapter(LocalDate.class, (JsonDeserializer<LocalDate>)
        (json, type, ctx) -> LocalDate.parse(json.getAsString()))
    .create();
```

---

### Problema: Biblioteca com reflexão legada (Kryo, XStream, etc.)

**✅ Adicionar `--add-opens` como medida paliativa temporária:**
```xml
<!-- maven-surefire-plugin ou configuração de JVM -->
<argLine>
  --add-opens java.base/java.io=ALL-UNNAMED
  --add-opens java.base/java.lang=ALL-UNNAMED
</argLine>
```
⚠️ Isso é uma correção temporária. A solução definitiva é atualizar a biblioteca ou substituir.

---

## Dependências Recomendadas para Java 21

| Biblioteca | Versão mínima para Java 21 |
|---|---|
| Gson | 2.10.1+ |
| Jackson | 2.15+ com `jackson-datatype-jsr310` |
| Mockito | 5.x+ |
| Hibernate | 6.x+ |
| Spring Boot | 3.x+ (Spring 6) |
| Kryo | 5.x+ |

---

## Flags JVM a Evitar (e suas alternativas)

| Flag legada | Status no Java 21 | Alternativa |
|---|---|---|
| `--illegal-access=permit` | **Removida** (era warning no 16, removida no 17) | Corrigir código ou usar `--add-opens` seletivo |
| `sun.misc.Unsafe` | Restrito | Use APIs públicas equivalentes |
| `setAccessible(true)` em campos JDK | **Bloqueado** | TypeAdapters, DTOs próprios |

---

## Template de Relatório de Auditoria

Após rodar os comandos de auditoria, documente:

```
## Auditoria de Migração Java X → Y

### Campos JDK em objetos serializados
- [ ] Nenhum encontrado
- [ ] Encontrados: [lista de arquivos/campos]

### Configurações de serialização
- [ ] Gson/Jackson com TypeAdapters para tipos JDK
- [ ] Sem uso de new Gson() sem configuração

### Flags JVM legadas
- [ ] Nenhuma flag --add-opens ou --illegal-access no projeto
- [ ] Encontradas: [lista]

### Bibliotecas em versões incompatíveis
- [ ] Todas compatíveis com Java 21
- [ ] Requer atualização: [lista]

### Status: ✅ Pronto para migrar / ⚠️ Ações necessárias antes
```

---

## Referências

- Leia `references/module-system.md` para entender o JPMS em profundidade
- Leia `references/gson-java21.md` para guia completo de migração do Gson para Java 21
