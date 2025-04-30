# Guía de configuración GitHub Actions Workflow - SonarQube

Esta guía proporciona instrucciones para integrar SonarQube en tu repositorio utilizando GitHub Actions, permitiendo análisis automáticos de calidad de código.

## ¿Qué contiene este repositorio plantilla?

Este repositorio incluye los archivos base necesarios para la integración con SonarQube:

1. **`.github/workflows/sonar.yml`**  
   Archivo principal del workflow de GitHub Actions encargado de ejecutar SonarScanner.

2. **`sonar-project.properties`**  
   Archivo de configuración que contiene los parámetros del proyecto para el análisis.

Los secretos necesarios (`SONAR_TOKEN` y `SONAR_HOST_URL`) ya están configurados a nivel de organización.

## Instrucciones básicas para desarrolladores

### Paso 1: Configura el archivo `sonar-project.properties`

El archivo `sonar-project.properties` ya existe en esta plantilla y solo necesitas personalizarlo:

```properties
# La clave del proyecto debe coincidir con el nombre del repositorio
sonar.projectKey=${GITHUB_REPOSITORY#*/}

# Nombre del proyecto (puede contener espacios y caracteres especiales)
sonar.projectName=Nombre de tu proyecto

# Versión del proyecto
sonar.projectVersion=1.0

# Ruta al código fuente (ajusta según la estructura de tu proyecto)
sonar.sources=src
```

Simplemente ajusta el `sonar.projectName` con un nombre descriptivo para tu proyecto y verifica que `sonar.sources` apunte correctamente a la carpeta donde está tu código fuente.

### Paso 2: Selecciona la configuración según tu lenguaje de programación

A continuación se muestran las configuraciones específicas para diferentes lenguajes de programación. Selecciona la que corresponda a tu proyecto y sigue las instrucciones.

## Configuración específica por lenguaje

> **Nota importante**: La configuración base de SonarQube ya soporta el análisis de muchos lenguajes de programación sin configuración adicional. Sin embargo, el uso de External Analyzers mejora significativamente la calidad y precisión del análisis, por lo que son altamente recomendados pero opcionales.

### Java (Configuración básica)

Para proyectos Java:

1. **Actualiza el archivo `sonar-project.properties`**:
```properties
sonar.projectKey=${GITHUB_REPOSITORY#*/}
sonar.projectName=Mi Proyecto Java
sonar.projectVersion=1.0

# Rutas específicas para Java
sonar.sources=src/main/java
sonar.java.binaries=target/classes
sonar.java.source=11

# Reportes de pruebas (si aplica)
sonar.junit.reportPaths=target/surefire-reports
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

2. **Modifica el archivo `.github/workflows/sonar.yml`** para incluir configuración Java:
```yaml
name: SonarQube Analysis
on:
  push:
    branches:
      - main
  pull_request:
    branches: 
      - main
jobs:
  sonarqube:
    name: SonarQube Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@master
        env:
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

#### Java con Spring Boot

Para proyectos Spring Boot, añade lo siguiente a la configuración básica de Java:

1. **Actualiza adicionalmente el archivo `sonar-project.properties`**:
```properties
# Exclusiones comunes en proyectos Spring Boot
sonar.exclusions=**/Application.java,**/config/**,**/model/**,**/dto/**
sonar.coverage.exclusions=**/model/**,**/dto/**,**/config/**
```

2. **Si utilizas Maven**, asegúrate de que tu pom.xml incluya la siguiente configuración:
```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.8</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>prepare-package</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

3. **Modifica el workflow para incluir la construcción y pruebas**:
```yaml
# Añade este paso después de setup-java y antes de SonarQube Scan
- name: Build and test with Maven
  run: mvn clean verify
```

### Python (Configuración básica)

Para proyectos Python:

1. **Actualiza el archivo `sonar-project.properties`**:
```properties
sonar.projectKey=${GITHUB_REPOSITORY#*/}
sonar.projectName=Mi Proyecto Python
sonar.projectVersion=1.0

# Rutas específicas para Python
sonar.sources=src
sonar.python.coverage.reportPaths=coverage.xml

# Configuración para Pylint
sonar.python.pylint.reportPaths=pylint-report.json
```

2. **Modifica el archivo `.github/workflows/sonar.yml`** para incluir la instalación de dependencias y la ejecución de Pylint:
```yaml
name: SonarQube Analysis
on:
  push:
    branches:
      - main
  pull_request:
    branches: 
      - main
jobs:
  sonarqube:
    name: SonarQube Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
          pip install pylint
      - name: Run Pylint
        run: pylint --output-format=json src/ > pylint-report.json || true
      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@master
        env:
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

#### Python con FastAPI

Para proyectos FastAPI, añade lo siguiente a la configuración básica de Python:

1. **Actualiza adicionalmente el archivo `sonar-project.properties`**:
```properties
# Rutas específicas para FastAPI
sonar.sources=app
sonar.exclusions=**/tests/**,**/migrations/**
sonar.coverage.exclusions=**/schemas/**,**/models/**,**/config.py
```

2. **Extiende el paso de instalación de dependencias**:
```yaml
- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install -r requirements.txt
    pip install pylint pytest pytest-cov
```

3. **Añade un paso para ejecutar pruebas con cobertura**:
```yaml
- name: Run tests with coverage
  run: |
    pytest --cov=app --cov-report=xml || true
```

### JavaScript/TypeScript (Configuración básica)

Para proyectos JavaScript o TypeScript:

1. **Actualiza el archivo `sonar-project.properties`**:
```properties
sonar.projectKey=${GITHUB_REPOSITORY#*/}
sonar.projectName=Mi Proyecto JavaScript
sonar.projectVersion=1.0

# Rutas específicas para JavaScript
sonar.sources=src
sonar.exclusions=**/*.test.js,**/*.spec.js,node_modules/**

# Reportes de ESLint y cobertura
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.eslint.reportPaths=eslint-report.json
```

2. **Modifica el archivo `.github/workflows/sonar.yml`** para incluir la instalación de Node.js y la ejecución de ESLint:
```yaml
name: SonarQube Analysis
on:
  push:
    branches:
      - main
  pull_request:
    branches: 
      - main
jobs:
  sonarqube:
    name: SonarQube Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Install dependencies
        run: npm ci
      - name: Run ESLint
        run: npx eslint src/ -f json -o eslint-report.json || true
      - name: Run tests with coverage
        run: npm test -- --coverage
        continue-on-error: true
      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@master
        env:
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

#### React con Vite

Para proyectos React con Vite, añade lo siguiente a la configuración básica de JavaScript/TypeScript:

1. **Actualiza adicionalmente el archivo `sonar-project.properties`**:
```properties
# Rutas específicas para React/Vite
sonar.exclusions=**/*.test.jsx,**/*.test.tsx,**/*.spec.jsx,**/*.spec.tsx,**/dist/**

# Configuración específica para TypeScript (si aplica)
sonar.typescript.tsconfigPath=tsconfig.json
```

2. **Actualiza el comando de ESLint para incluir extensiones JSX/TSX**:
```yaml
- name: Run ESLint
  run: |
    npx eslint --ext .jsx,.js,.tsx,.ts src/ -f json -o eslint-report.json || true
```

### C/C++ (Configuración básica)

Para proyectos C/C++:

1. **Actualiza el archivo `sonar-project.properties`**:
```properties
sonar.projectKey=${GITHUB_REPOSITORY#*/}
sonar.projectName=Mi Proyecto C++
sonar.projectVersion=1.0

# Rutas específicas para C/C++
sonar.sources=src
sonar.exclusions=tests/**/*

# Configuración de Build Wrapper
sonar.cfamily.build-wrapper-output=bw-output
```

2. **Modifica el archivo `.github/workflows/sonar.yml`** para incluir el Build Wrapper:
```yaml
name: SonarQube Analysis
on:
  push:
    branches:
      - main
  pull_request:
    branches: 
      - main
jobs:
  sonarqube:
    name: SonarQube Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Install build essentials
        run: sudo apt-get install -y build-essential
      - name: Download Build Wrapper
        run: |
          mkdir -p $HOME/.sonar
          curl -sSLo $HOME/.sonar/build-wrapper-linux-x86.zip https://sonarcloud.io/static/cpp/build-wrapper-linux-x86.zip
          unzip -o $HOME/.sonar/build-wrapper-linux-x86.zip -d $HOME/.sonar/
      - name: Build with Build Wrapper
        run: |
          $HOME/.sonar/build-wrapper-linux-x86/build-wrapper-linux-x86-64 --out-dir bw-output make clean all
      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@master
        env:
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### Configuraciones para otros frameworks

Además de los frameworks mencionados anteriormente, SonarQube puede analizar proyectos desarrollados con cualquier otro framework. Para ello, utiliza la configuración base del lenguaje correspondiente y ajusta las rutas y exclusiones según sea necesario.

## Consejos y buenas prácticas

- **Estructura de código**: Organiza tu código en carpetas lógicas para facilitar la configuración de `sonar.sources` y `sonar.exclusions`.

- **Operador `|| true`**: Este operador se utiliza después de comandos como Pylint o ESLint para que el workflow continúe incluso si hay errores o advertencias en el análisis.

- **Exclusiones**: Utiliza `sonar.exclusions` para excluir archivos que no deben analizarse:
  ```properties
  # Excluir archivos de test, de terceros, o generados automáticamente
  sonar.exclusions=**/*.test.js,**/node_modules/**,**/generated/**
  ```
## Usando SonarLint - Plugin para IDE

SonarLint es una extensión para tu IDE que te permite detectar y corregir problemas de calidad de código mientras programas, antes de enviar tus cambios a SonarQube.

### ¿Por qué usar SonarLint?

- **Detección temprana**: Encuentra problemas mientras escribes código, sin esperar al análisis del CI/CD
- **Desarrollo más rápido**: Corrige problemas inmediatamente sin ciclos de retroalimentación largos
- **Coherencia con SonarQube**: Usa las mismas reglas que el análisis en servidor
- **Modo conectado**: Sincroniza con el servidor de SonarQube para mantener configuraciones consistentes

### IDEs soportados

SonarLint está disponible para:
- IntelliJ IDEA (y otros productos JetBrains)
- Visual Studio Code
- Eclipse
- Visual Studio

### Instalación

#### Para Visual Studio Code:
1. Abre VS Code
2. Ve a la pestaña de extensiones (Ctrl+Shift+X o Cmd+Shift+X)
3. Busca "SonarLint"
4. Haz clic en "Instalar"

#### Para IntelliJ IDEA/WebStorm/PyCharm:
1. Ve a Settings/Preferences > Plugins
2. Busca "SonarLint" en el Marketplace
3. Haz clic en "Instalar"
4. Reinicia el IDE cuando se te solicite

#### Para Eclipse:
1. Ayuda > Eclipse Marketplace
2. Busca "SonarLint"
3. Haz clic en "Instalar"

### Configuración del modo conectado

Para sincronizar SonarLint con nuestro servidor de SonarQube:

#### En VS Code:
1. Abre la paleta de comandos (Ctrl+Shift+P o Cmd+Shift+P)
2. Escribe "SonarLint: Connect to SonarQube"
3. Completa la configuración:
   - URL del servidor: `https://sonarqube.ia.nuvu.cc`
   - Token: Genera uno personal desde SonarQube > Mi cuenta > Tokens de seguridad
   - Proyecto: Selecciona tu proyecto

#### En IntelliJ IDEA:
1. Ve a Settings/Preferences > Tools > SonarLint
2. Haz clic en "+" para añadir una conexión a SonarQube
3. Completa la configuración con la URL y el token
4. En la pestaña "Project Settings", asocia tu proyecto local con el proyecto de SonarQube

### Uso diario

Una vez instalado, SonarLint:
- Subrayará problemas potenciales directamente en tu editor
- Proporcionará detalles sobre el problema al pasar el cursor sobre el subrayado
- Ofrecerá acciones rápidas (quickfixes) para algunos problemas
- Mostrará la gravedad de los problemas (crítico, mayor, menor, etc.)

### Beneficios para el equipo

- **Calidad consistente**: Todos los miembros del equipo siguen las mismas reglas
- **Menor revisión de código**: Menos comentarios sobre problemas básicos de calidad en las revisiones
- **Mejor Code Quality Gate**: Menos probabilidades de fallar en los análisis de SonarQube
- **Aprendizaje continuo**: Los desarrolladores aprenden las mejores prácticas mientras programan

### Personalización

Puedes ajustar reglas específicas:
1. Ve a la configuración de SonarLint en tu IDE
2. Busca las reglas por lenguaje
3. Habilita, deshabilita o ajusta la severidad según las necesidades de tu proyecto

> **Nota**: En modo conectado, las reglas se sincronizarán con el servidor SonarQube y algunas personalizaciones locales podrían ser sobrescritas.


## Solución de problemas comunes

### El análisis falla con errores de compilación

- Verifica que tu proyecto compile correctamente localmente antes de hacer push.
- Asegúrate de que todas las dependencias estén correctamente especificadas en tus archivos (pom.xml, requirements.txt, package.json).

### Errores en la detección de idioma

- Asegúrate de que `sonar.sources` apunte a la carpeta correcta donde está tu código fuente.
- Verifica que los archivos tengan la extensión correcta (.java, .py, .js, .cpp, etc.).

## Recursos adicionales

Para más información sobre SonarQube y las reglas de calidad, consulta:
- [Documentación oficial de SonarQube](https://docs.sonarqube.org/)
- [Reglas de calidad por lenguaje](https://rules.sonarsource.com/)
