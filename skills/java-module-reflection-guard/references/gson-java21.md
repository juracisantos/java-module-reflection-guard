# Gson e Java 21 — Guia Completo de Migração

## O problema raiz

O Gson usa `ReflectionFactory` e `Field.setAccessible(true)` para acessar campos privados de qualquer classe. No Java 21, campos de módulos não exportados (como `java.base`) **não podem ser acessados**, mesmo com `setAccessible(true)`.

## Diagnóstico rápido

Se você vê qualquer um desses erros, este guia se aplica:
- `Failed making field 'java.io.File#path' accessible`
- `InaccessibleObjectException: Unable to make field private java.lang.String java.io.File.path accessible`
- `module java.base does not open java.io to unnamed module`

## Solução definitiva: GsonBuilder com TypeAdapters

```java
import com.google.gson.*;
import java.io.File;
import java.lang.reflect.Type;
import java.time.*;
import java.net.URL;

@Configuration
public class GsonConfig {

    @Bean
    public Gson gson() {
        return new GsonBuilder()
            // File → serializa como path string
            .registerTypeHierarchyAdapter(File.class, new FileTypeAdapter())
            // Tipos java.time
            .registerTypeAdapter(LocalDate.class, new LocalDateAdapter())
            .registerTypeAdapter(LocalDateTime.class, new LocalDateTimeAdapter())
            .registerTypeAdapter(Instant.class, new InstantAdapter())
            // Campos sem @Expose ficam visíveis (comportamento padrão)
            .serializeNulls() // remova se não quiser serializar nulls
            .create();
    }

    // --- Adapters ---

    static class FileTypeAdapter implements JsonSerializer<File>, JsonDeserializer<File> {
        @Override
        public JsonElement serialize(File src, Type type, JsonSerializationContext ctx) {
            return src == null ? JsonNull.INSTANCE : new JsonPrimitive(src.getAbsolutePath());
        }
        @Override
        public File deserialize(JsonElement json, Type type, JsonDeserializationContext ctx) {
            return json.isJsonNull() ? null : new File(json.getAsString());
        }
    }

    static class LocalDateAdapter implements JsonSerializer<LocalDate>, JsonDeserializer<LocalDate> {
        @Override
        public JsonElement serialize(LocalDate src, Type type, JsonSerializationContext ctx) {
            return new JsonPrimitive(src.toString()); // ISO-8601: "2024-01-15"
        }
        @Override
        public LocalDate deserialize(JsonElement json, Type type, JsonDeserializationContext ctx) {
            return LocalDate.parse(json.getAsString());
        }
    }

    static class LocalDateTimeAdapter implements JsonSerializer<LocalDateTime>, JsonDeserializer<LocalDateTime> {
        @Override
        public JsonElement serialize(LocalDateTime src, Type type, JsonSerializationContext ctx) {
            return new JsonPrimitive(src.toString());
        }
        @Override
        public LocalDateTime deserialize(JsonElement json, Type type, JsonDeserializationContext ctx) {
            return LocalDateTime.parse(json.getAsString());
        }
    }

    static class InstantAdapter implements JsonSerializer<Instant>, JsonDeserializer<Instant> {
        @Override
        public JsonElement serialize(Instant src, Type type, JsonSerializationContext ctx) {
            return new JsonPrimitive(src.toString());
        }
        @Override
        public Instant deserialize(JsonElement json, Type type, JsonDeserializationContext ctx) {
            return Instant.parse(json.getAsString());
        }
    }
}
```

## Estratégias para DTOs com campos problemáticos

### Estratégia A — DTO sem tipos JDK internos (mais segura)
```java
// Antes (problemático)
public class EnvioEmail {
    private String destinatario;
    private List<File> anexos; // ← problema
}

// Depois — DTO limpo
public class EnvioEmail {
    private String destinatario;
    private List<String> caminhoAnexos; // apenas o path como String

    // Método auxiliar para obter Files quando necessário
    public List<File> getAnexosComoFile() {
        return caminhoAnexos.stream().map(File::new).collect(Collectors.toList());
    }
}
```

### Estratégia B — Excluir da serialização com @Expose
```java
@Bean
public Gson gson() {
    return new GsonBuilder()
        .excludeFieldsWithoutExposeAnnotation()
        .create();
}

public class EnvioEmail {
    @Expose
    private String destinatario;

    // Não tem @Expose → não serializado, não causa erro
    private List<File> anexos;
}
```

### Estratégia C — ExclusionStrategy customizada
```java
@Bean
public Gson gson() {
    return new GsonBuilder()
        .addSerializationExclusionStrategy(new ExclusionStrategy() {
            @Override
            public boolean shouldSkipField(FieldAttributes f) {
                // Pula qualquer campo cujo tipo é java.io.File ou subclasse
                return File.class.isAssignableFrom(f.getDeclaredClass());
            }
            @Override
            public boolean shouldSkipClass(Class<?> clazz) {
                return false;
            }
        })
        .create();
}
```

## Versão do Gson

Use **Gson 2.10.1 ou superior** com Java 21. Versões anteriores não têm as correções para o module system.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.10.1</version>
</dependency>
```
