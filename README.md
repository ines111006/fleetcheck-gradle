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

## Evidência 1

Linha de erro:
[ERROR] App.java:[4,38] package com.fasterxml.jackson.databind does not exist

Import que causou o erro: import com.fasterxml.jackson.databind.ObjectMapper; (linha 4 do App.java).
O mesmo acontece com import com.fasterxml.jackson.core.type.TypeReference; (linha 3).
O App.java usa a biblioteca Jackson, mas o pom.xml não a declara como dependência, por isso o Maven não a descarrega e o compilador não encontra as classes ObjectMapper e TypeReference.
## Passo 2

Depois de adicionar a dependência jackson-databind ao pom.xml, o projeto passou a compilar (BUILD SUCCESS).

Nota: o projeto fornecido não tinha pasta src/test, por isso o Maven não tinha testes para executar e a falha na fase de testes descrita na ficha não aconteceu. O defeito existia no FleetService.java: a condição usava ">" em vez de ">=". Um veículo com exatamente 10000 km desde a última revisão (V3) não era contado como precisando de revisão. Corrigi para ">=".

Pergunta: Porque é que esta falha é melhor do que a do passo 1?
No passo 1 o build parava na compilação por um problema de configuração (dependência em falta), por isso nem era possível verificar se o programa funcionava. Uma falha nos testes significa que o build já avançou mais: o código compila e as dependências estão resolvidas, e o que falha agora é o comportamento do programa. É uma falha mais útil, porque aponta para um defeito real na lógica da aplicação e não para um problema de configuração.
## Passo 3

Resultado de mvn dependency:tree:
- Dependência direta: com.fasterxml.jackson.core:jackson-databind:2.22.2 (compile)
- Dependências transitivas (trazidas pelo jackson-databind): jackson-core:2.22.2 e jackson-annotations:2.22 (compile)
- junit-jupiter:5.14.4 (test) e as suas dependências transitivas, usadas apenas nos testes.

## Evidência 4

O JAR por defeito (target/fleetcheck-1.0.0.jar) não era executável. Ao correr java -jar apareceu o erro "no main manifest attribute", porque o ficheiro MANIFEST.MF não indicava a classe principal. Além disso, esse JAR só continha as classes do projeto, sem o Jackson, por isso mesmo com a classe principal definida o programa falharia ao precisar das bibliotecas.

O plugin Shade criou um segundo JAR, target/fleetcheck-1.0.0-all.jar (um "uber JAR"), que:
- tem no MANIFEST.MF a entrada Main-Class: pt.upt.fleetcheck.App (definida pelo ManifestResourceTransformer);
- inclui dentro dele as classes das dependências de runtime (jackson-databind, jackson-core e jackson-annotations).

Assim o JAR passa a ser autossuficiente e pode ser executado com java -jar target/fleetcheck-1.0.0-all.jar, produzindo o resultado esperado:
FleetCheck 1.0 | Vehicles loaded: 4 | Vehicles requiring service: 2 | Average mileage: 37000 km

## Passo 5

Foi gerado o Maven Wrapper com mvn wrapper:wrapper. O projeto passou a conter mvnw, mvnw.cmd e a pasta .mvn/wrapper/, configurada para usar o Maven 3.10.0. A partir daqui o build é feito com mvnw.cmd clean verify.

Foi também adicionada a propriedade project.build.outputTimestamp (2026-09-27T00:00:00Z), para que os plugins compatíveis usem sempre a mesma data nos ficheiros gerados e o mesmo código produza sempre o mesmo artefacto.

Pergunta: Que pressuposto escondido sobre o ambiente foi eliminado pelo wrapper?
O pressuposto de que quem compila o projeto já tem o Maven instalado e configurado no PATH, e na versão certa. Sem o wrapper, o build dependia da máquina do programador: noutra máquina o Maven podia não existir ou ser de outra versão, e o resultado podia ser diferente. Com o wrapper, a versão do Maven fica definida dentro do próprio projeto e é descarregada automaticamente se for necessário.

## Passo 6

O projeto foi enviado para o GitHub (https://github.com/ines111006/fleetcheck-maven) e foi criado o workflow .github/workflows/build.yml. Em cada push ou pull request, o GitHub Actions usa uma máquina Ubuntu com Java 21, executa ./mvnw -B clean verify e guarda o JAR gerado como artefacto (fleetcheck-build).

Execução com sucesso: https://github.com/ines111006/fleetcheck-maven/actions/runs/37235774588

O ficheiro mvnw foi marcado como executável antes do primeiro commit (git update-index --chmod=+x mvnw), para evitar o erro de permissões no Linux.

## Evidência 7

Foi adicionado o plugin cyclonedx-maven-plugin (versão 2.9.3), executado na fase verify. O comando mvnw.cmd clean verify gerou o ficheiro target/bom.json, um SBOM no formato CycloneDX 1.6 com 3 componentes: jackson-databind 2.22.2, jackson-core 2.22.2 e jackson-annotations 2.22.

Porque é que o SBOM contém componentes que não foram escritos explicitamente no pom.xml?
No pom.xml só foi declarada a dependência direta jackson-databind. No entanto, o jackson-databind precisa do jackson-core e do jackson-annotations para funcionar, por isso o Maven resolve automaticamente essas dependências transitivas (como se viu no mvn dependency:tree). O SBOM descreve o que realmente vai dentro do software, e não só o que foi escrito no pom.xml, por isso inclui todo o grafo de dependências de runtime: diretas e transitivas. As dependências de teste (JUnit) não aparecem porque não fazem parte do produto final.

# Parte Gradle (FleetCheck_Gradle)

## Evidência 8.1

Linha de erro:
Execution failed for task ':compileJava'

O build.gradle inicial não declarava o Jackson. Como o App.java importa com.fasterxml.jackson.databind.ObjectMapper e com.fasterxml.jackson.core.type.TypeReference, a compilação falhou. A dependência em falta é com.fasterxml.jackson.core:jackson-databind:2.22.2, tal como aconteceu no passo 1 com o Maven.

## Evidência 8.2

Depois de adicionar implementation 'com.fasterxml.jackson.core:jackson-databind:2.22.2' ao build.gradle, o comando gradle clean build terminou com BUILD SUCCESSFUL.

Resultado de gradle dependencies --configuration runtimeClasspath:
- Dependência direta: com.fasterxml.jackson.core:jackson-databind:2.22.2
- Dependências transitivas: jackson-core:2.22.2 e jackson-annotations:2.22
- Aparece também o jackson-bom:2.22.2, usado pelo Jackson para alinhar as versões dos seus módulos (as entradas marcadas com (c) são restrições de versão).

Comparação com mvn dependency:tree: o resultado é o mesmo. Mudar o sistema de build não mudou as dependências da aplicação. A dependência direta, as dependências transitivas e as versões são iguais em Maven e Gradle; só muda a forma de as declarar (bloco <dependency> no pom.xml vs uma linha implementation no build.gradle) e de as visualizar.

## Evidência 8.3

O JAR por defeito gerado por gradle clean jar (build/libs/fleetcheck-1.0.0.jar) não era executável: java -jar devolvia "no main manifest attribute", porque o manifest não indicava a classe principal e o JAR só continha as classes do projeto, sem o Jackson.

Foram adicionados ao build.gradle o plugin application (com mainClass = 'pt.upt.fleetcheck.App') e uma configuração da tarefa jar que:
- escreve no manifest a entrada Main-Class: pt.upt.fleetcheck.App;
- copia para dentro do JAR o conteúdo de todas as dependências de runtime (configurations.runtimeClasspath), ou seja, jackson-databind, jackson-core e jackson-annotations;
- usa DuplicatesStrategy.EXCLUDE para ignorar ficheiros repetidos entre bibliotecas.

O JAR passou a ser autossuficiente (equivalente ao fat JAR criado pelo plugin Shade no Maven) e java -jar build/libs/fleetcheck-1.0.0.jar produziu:
FleetCheck 1.0 | Vehicles loaded: 4 | Vehicles requiring service: 2 | Average mileage: 37000 km

## Passo 8.4

Foi gerado o Gradle Wrapper com gradle wrapper. O projeto passou a conter gradlew, gradlew.bat e a pasta gradle/wrapper/. O build passou a ser feito com gradlew.bat clean build, que terminou com BUILD SUCCESSFUL.

Pergunta: Que pressuposto escondido sobre o ambiente foi eliminado pelo Gradle Wrapper?
O pressuposto de que quem compila o projeto tem o Gradle instalado e configurado no PATH, na versão certa. Com o wrapper, a versão do Gradle fica definida no próprio projeto (gradle/wrapper/gradle-wrapper.properties) e é descarregada automaticamente se não existir, por isso o build funciona da mesma forma em qualquer máquina, incluindo o GitHub Actions.

## Evidência 8.5

O projeto Gradle foi enviado para https://github.com/ines111006/fleetcheck-gradle e foi criado o workflow .github/workflows/build-gradle.yml. Em cada push ou pull request, o GitHub Actions usa uma máquina Ubuntu 24.04 com Java 21, executa ./gradlew clean build e guarda o JAR de build/libs como artefacto (fleetcheck-gradle-build). O gradlew foi marcado como executável antes do commit (git update-index --chmod=+x gradlew).

Execução com sucesso: https://github.com/ines111006/fleetcheck-gradle/actions/runs/37239236606

## Evidência 8.6

Foi adicionado o plugin org.cyclonedx.bom (versão 3.4.1) ao build.gradle. O comando gradlew.bat cyclonedxBom gerou o ficheiro build/reports/cyclonedx/bom.json, que contém jackson-databind 2.22.2, jackson-core 2.22.2 e jackson-annotations 2.22.

Porque é que o SBOM do Gradle contém dependências que não foram escritas no build.gradle?
No build.gradle só foi declarada a dependência direta jackson-databind. O Gradle resolve automaticamente as dependências transitivas de que ela precisa (jackson-core e jackson-annotations), como se viu em gradle dependencies. O SBOM descreve o que realmente faz parte da aplicação em runtime, e não só o que foi escrito no ficheiro de build, por isso inclui todo o grafo de dependências: diretas e transitivas.

## 8.7 Comparação Maven vs Gradle

| Tarefa | Maven | Gradle |
|---|---|---|
| Configuração do build | pom.xml | build.gradle (+ settings.gradle) |
| Build limpo | mvnw.cmd clean verify | gradlew.bat clean build |
| Adicionar dependência | bloco <dependency>...</dependency> | implementation 'group:artifact:version' |
| Ver dependências | mvn dependency:tree | gradle dependencies --configuration runtimeClasspath |
| Wrapper | mvnw / mvnw.cmd + .mvn/wrapper | gradlew / gradlew.bat + gradle/wrapper |
| Saída do build | target/ | build/ |
| Localização do JAR | target/ (fleetcheck-1.0.0-all.jar) | build/libs/ (fleetcheck-1.0.0.jar) |
| JAR executável | plugin maven-shade-plugin | plugin application + configuração da tarefa jar |
| SBOM | CycloneDX Maven plugin → target/bom.json | CycloneDX Gradle plugin → build/reports/cyclonedx/bom.json |
| CI | .github/workflows/build.yml | .github/workflows/build-gradle.yml |

Pergunta final: Maven e Gradle construíram exatamente a mesma aplicação FleetCheck. O que mudou: o software ou o processo de build?
Mudou apenas o processo de build. O código-fonte, os recursos, as dependências (diretas e transitivas, com as mesmas versões) e o resultado da execução (FleetCheck 1.0 | Vehicles loaded: 4 | Vehicles requiring service: 2 | Average mileage: 37000 km) foram iguais nas duas ferramentas. O que mudou foi a forma de descrever e executar o build: o formato do ficheiro de configuração, os comandos, as pastas de saída e os plugins usados para gerar o JAR executável e o SBOM.

