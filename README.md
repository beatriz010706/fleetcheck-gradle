# FleetCheck – Build Systems Lab

This project is intentionally incomplete. Follow the worksheet in the order given.

Expected final application output:

```
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km
```

Do not copy the solution POM. The objective is to observe how each build change alters the result.

## Relatório — FleetCheck (Gradle)

### Passo 8.1 — Build inicial (falha esperada)
Projeto Gradle criado a partir da cópia do código-fonte do projeto Maven (`src/` completo), com `build.gradle` e `settings.gradle` novos, sem a dependência do Jackson.

Erro de compilação obtido ao correr `gradle clean build`:

App.java:3: error: package com.fasterxml.jackson.core.type does not exist
import com.fasterxml.jackson.core.type.TypeReference;

**Dependência em falta:** `com.fasterxml.jackson.core:jackson-databind` — tal como na versão Maven, o `build.gradle` inicial não a declarava.

### Passo 8.2 — Dependência do Jackson e grafo de dependências
Após adicionar `implementation 'com.fasterxml.jackson.core:jackson-databind:2.22.2'`, a compilação passou com sucesso.

Inspeção com `gradle dependencies --configuration runtimeClasspath`:
- `jackson-databind:2.22.2` aparece como dependência **direta**.
- `jackson-core` e `jackson-annotations` aparecem como dependências **transitivas**, indentadas por baixo de `jackson-databind`.

Comparação com `mvn dependency:tree`: o resultado é equivalente — as mesmas três bibliotecas, com a mesma relação direta/transitiva. Mudar a ferramenta de build não alterou as dependências da aplicação, apenas a forma de as declarar e inspecionar.

**Nota:** tal como no Maven, o projeto não contém testes em `src/test/java`, pelo que a fase de testes não teve nada para executar em nenhuma das duas ferramentas.

### Passo 8.3 — JAR executável
- JAR normal (`fleetcheck-1.0.0.jar`, criado com `gradle clean jar`): não é executável de forma autónoma, por não incluir `Main-Class` nem as dependências.
- Após configurar o plugin `application` (com `mainClass = 'pt.upt.fleetcheck.App'`) e o bloco `jar { ... }` a incluir o `runtimeClasspath`, o JAR passou a ser um "fat JAR" executável.

Saída obtida:

FleetCheck 1.0 | Vehicles loaded: 4 | Vehicles requiring service: 1 | Average mileage: 37000 km

Resultado idêntico ao obtido na versão Maven, incluindo o mesmo defeito de comportamento ("Vehicles requiring service: 1" em vez do esperado "2"), confirmando que o defeito está no código-fonte e não é introduzido pela ferramenta de build.

### Passo 8.4 — Gradle Wrapper
Gerado com `gradle wrapper`, criando/atualizando `gradlew`, `gradlew.bat` e `gradle/wrapper/`.

**Pergunta:** o Gradle Wrapper remove a suposição de que quem for correr o build (colega ou CI) já tem o Gradle instalado na máquina, com a versão correta. O wrapper descarrega automaticamente a versão certa do Gradle na primeira execução, tornando o build reprodutível em qualquer ambiente — o mesmo papel desempenhado pelo Maven Wrapper na parte anterior.

Testado localmente com sucesso:

.\gradlew.bat clean build


### Passo 8.5 — GitHub Actions
Novo repositório criado em `https://github.com/beatriz010706/fleetcheck-gradle`, com workflow em `.github/workflows/build-gradle.yml`, correndo `./gradlew clean build` e fazendo upload do JAR como artefacto.

Foi necessário tornar o `gradlew` executável antes do push, por o Windows não preservar essa permissão ao fazer commit:

git update-index --chmod=+x gradlew


Execução com sucesso: <URL_DO_RUN>

### Passo 8.6 — SBOM (CycloneDX Gradle plugin)
Adicionado o plugin `id 'org.cyclonedx.bom' version '3.4.1'` ao `build.gradle`. Gerado com:

.\gradlew.bat cyclonedxBom

Ficheiro `build/reports/cyclonedx/bom.json` gerado com sucesso, contendo `jackson-databind`, `jackson-core` e `jackson-annotations`.

**Pergunta:** o SBOM contém `jackson-core` e `jackson-annotations` mesmo sem terem sido declarados explicitamente no `build.gradle`, porque são dependências transitivas, resolvidas automaticamente a partir de `jackson-databind`. O SBOM reflete o grafo completo de dependências usado em runtime, não apenas as declaradas diretamente — a mesma razão apontada no SBOM da versão Maven.

### Passo 8.7 — Comparação Maven vs Gradle

| Task | Maven | Gradle |
|---|---|---|
| Build configuration | pom.xml | build.gradle |
| Clean build | mvnw.cmd clean verify | gradlew.bat clean build |
| Add dependency | `<dependency>...</dependency>` | `implementation 'group:artifact:version'` |
| Inspect dependencies | mvn dependency:tree | gradle dependencies |
| Wrapper | mvnw.cmd | gradlew.bat |
| Build output | target/ | build/ |
| JAR location | target/ | build/libs/ |
| SBOM | CycloneDX Maven plugin | CycloneDX Gradle plugin |

**Pergunta final:** *Both Maven and Gradle built exactly the same FleetCheck application. What changed: the software or the build process?*

O software manteve-se exatamente o mesmo — o mesmo código-fonte, as mesmas dependências (Jackson, JUnit) e o mesmo resultado final, incluindo o mesmo defeito de comportamento. O que mudou foi exclusivamente o **processo de build**: a forma de declarar dependências (XML vs DSL Groovy), as ferramentas de compilação, empacotamento e geração de relatórios, e a sintaxe dos comandos. Isto demonstra que o build system é uma camada independente da lógica da aplicação — diferentes ferramentas podem produzir o mesmo artefacto funcional a partir do mesmo código-fonte.
