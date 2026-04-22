# java-module-reflection-guard

Skill para Claude Code que detecta, previne e corrige problemas de compatibilidade ao migrar entre versões do Java, com foco no **Java 9+ Module System (JPMS)** e no **strong encapsulation** que afeta serialização, reflexão e bibliotecas de terceiros.

## O problema que ela resolve

A partir do Java 17 (e especialmente no Java 21), o runtime **bloqueia ativamente** acesso via reflexão a campos internos de classes do JDK. Isso causa erros como:

```
Failed making field 'java.io.File#path' accessible
InaccessibleObjectException: Unable to make field private ... accessible
module java.base does not open java.io to unnamed module
```

Esses erros frequentemente só aparecem em produção — quando dados que estavam `null` em dev passam a ser instanciados, ou quando flags `--add-opens` herdadas deixam de existir no novo ambiente.

## O que a skill cobre

- Checklist de auditoria com comandos bash para rodar antes de migrar
- Categorias de problemas: Gson/Jackson com tipos JDK, reflexão em campos privados, bibliotecas legadas
- Soluções concretas com antes/depois de código
- Tabela de versões mínimas compatíveis com Java 21
- Template de relatório de auditoria

## Referências incluídas

| Arquivo | Conteúdo |
|---|---|
| `skills/java-module-reflection-guard/references/gson-java21.md` | Guia completo de migração do Gson para Java 21, com `GsonConfig` completo e 4 TypeAdapters |
| `skills/java-module-reflection-guard/references/module-system.md` | Referência técnica do JPMS: linha do tempo, pacotes bloqueados, `--add-opens`, ferramentas de detecção |

## Instalação via Marketplace (recomendado)

**1. Registrar o marketplace** — adicione em `~/.claude/settings.json`:

```json
"extraKnownMarketplaces": {
  "juracisantos-skills": {
    "source": {
      "source": "github",
      "repo": "juracisantos/java-module-reflection-guard"
    }
  }
}
```

**2. Instalar o plugin** — numa sessão do Claude Code:

```
/plugin install java-module-reflection-guard@juracisantos-skills
```

## Instalação manual (alternativa)

```bash
git clone https://github.com/juracisantos/java-module-reflection-guard ~/.claude/skills/java-module-reflection-guard
```

Após clonar, a skill é detectada automaticamente pelo Claude Code na próxima sessão.
