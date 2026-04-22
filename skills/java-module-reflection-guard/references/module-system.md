# Java Module System (JPMS) — Referência para Migrações

## Linha do tempo de encapsulamento

| Versão | Comportamento |
|---|---|
| Java 8 | Reflexão livre, acesso a qualquer campo via setAccessible(true) |
| Java 9-11 | JPMS introduzido, --illegal-access=permit como padrão (warning) |
| Java 16 | --illegal-access gera warning mais visível, foi deprecated |
| Java 17 | --illegal-access=deny por padrão. Acesso bloqueado com exceção |
| Java 21 | Encapsulamento forte. InaccessibleObjectException sem exceção |

## Por que `java.io.File` é afetado

`java.io.File` faz parte do módulo `java.base`. Este módulo **não exporta** o pacote `java.io` para módulos sem nome (unnamed modules — onde fica o código legacy sem `module-info.java`).

Quando o Gson faz:
```java
Field field = File.class.getDeclaredField("path");
field.setAccessible(true); // ← InaccessibleObjectException aqui
```

O runtime lança exceção porque `java.base/java.io` não está aberto para reflexão.

## Quais pacotes do JDK são afetados

Principais pacotes bloqueados para reflexão profunda:
- `java.io` → `File`, `InputStream`, `OutputStream`
- `java.lang` → `String` (campos internos), `Thread`
- `java.lang.reflect`
- `java.net` → `URL`, `URI`, `Socket`
- `java.nio` → `Path`, `ByteBuffer`
- `java.time` → toda a API de datas
- `java.util` → coleções internas, `Optional`
- `sun.*`, `com.sun.*` → APIs internas (sempre foram privadas, agora inacessíveis)

## Como `--add-opens` funciona (workaround temporário)

```
--add-opens <módulo>/<pacote>=<módulo-destino>
```

Exemplo para desbloquear java.io para reflexão:
```
--add-opens java.base/java.io=ALL-UNNAMED
```

**Onde configurar:**
```xml
<!-- Maven Surefire (testes) -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>--add-opens java.base/java.io=ALL-UNNAMED</argLine>
    </configuration>
</plugin>

<!-- Spring Boot Maven Plugin (aplicação) -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <jvmArguments>--add-opens java.base/java.io=ALL-UNNAMED</jvmArguments>
    </configuration>
</plugin>
```

```yaml
# application.yml / docker / kubernetes
JAVA_OPTS: "--add-opens java.base/java.io=ALL-UNNAMED"
```

⚠️ **Isso é paliativo.** A solução correta é eliminar a dependência de reflexão em tipos JDK.

## Como detectar problemas antes de ir para produção

### Teste de integração com serialização real
```java
@Test
void deveLancarExcecaoSeObjetoNaoForSerializavel() {
    EnvioEmail email = new EnvioEmail();
    email.setAnexos(List.of(new File("/tmp/test.pdf"))); // instanciado, não null
    
    assertDoesNotThrow(() -> gson.toJson(email),
        "Serialização deve funcionar com Java 21");
}
```

### Ferramenta: jdeps (análise de dependências de módulos)
```bash
# Analisa dependências do seu JAR em relação ao module system
jdeps --multi-release 21 --print-module-deps target/meu-app.jar

# Verifica uso de APIs internas
jdeps --jdk-internals target/meu-app.jar
```

### Ferramenta: jlink (verifica compatibilidade)
```bash
jlink --module-path $JAVA_HOME/jmods --add-modules java.base --output /tmp/test-runtime
```
