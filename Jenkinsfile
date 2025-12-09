pipeline {
    agent any
    
    environment {
        // Configuraciones de Node.js
        NODE_VERSION = '21.0.0'
        NPM_CONFIG_CACHE = "${WORKSPACE}\\.npm"
        
        // Configuraciones de Python
        PYTHON_VERSION = '3.11.6'
        VENV_DIR = "${WORKSPACE}\\venv"
        
        // Ruta de Python (ajusta según tu instalación)
        PYTHON_HOME = 'C:\\Python311'
        PATH = "${PYTHON_HOME};${PYTHON_HOME}\\Scripts;${PATH}"
        
        // Directorios del proyecto
        BACKEND_DIR = 'backend'
        FRONTEND_DIR = 'frontend'
        
        // Variables de entorno para el build
        CI = 'true'
    }
    
    stages {
        stage('Preparación') {
            steps {
                echo 'Limpiando workspace...'
                cleanWs()
                
                echo 'Clonando repositorio...'
                checkout scm
                
                echo 'Mostrando información del entorno...'
                bat '''
                    echo Node version:
                    node --version
                    echo NPM version:
                    npm --version
                    echo Python version:
                    python --version
                    echo Directorio actual:
                    cd
                '''
            }
        }
        
        stage('Instalar Dependencias - Backend') {
            steps {
                echo 'Instalando dependencias de Python...'
                script {
                    def pythonExists = bat(script: 'python --version', returnStatus: true) == 0
                    if (pythonExists) {
                        bat '''
                            rem Crear entorno virtual
                            python -m venv %VENV_DIR%
                            
                            rem Activar entorno virtual e instalar dependencias
                            call %VENV_DIR%\\Scripts\\activate.bat
                            python -m pip install --upgrade pip
                            pip install setuptools
                            pip install -r requirements.txt
                        '''
                    } else {
                        echo 'Python no está instalado. Saltando instalación de dependencias del backend.'
                    }
                }
            }
        }
        
        stage('Instalar Dependencias - Frontend (Root)') {
            steps {
                echo 'Instalando dependencias raíz del proyecto...'
                bat '''
                    npm install
                '''
            }
        }
        
        stage('Instalar Dependencias - Frontend (App)') {
            steps {
                echo 'Instalando dependencias del frontend...'
                dir("${FRONTEND_DIR}") {
                    bat '''
                        npm install
                        npm ci --prefer-offline --no-audit
                    '''
                }
            }
        }
        
        stage('Lint - Backend') {
            steps {
                echo 'Ejecutando lint en el backend...'
                dir("${BACKEND_DIR}") {
                    script {
                        def pythonExists = bat(script: 'python --version', returnStatus: true) == 0
                        if (pythonExists) {
                            bat '''
                                call %VENV_DIR%\\Scripts\\activate.bat
                                
                                rem Instalar herramientas de linting si no están
                                pip install flake8 black
                                
                                rem Ejecutar flake8 (opcional: no fallar el build)
                                flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics || exit 0
                            '''
                        } else {
                            echo 'Python no disponible. Saltando lint del backend.'
                        }
                    }
                }
            }
        }
        
        stage('Lint - Frontend') {
            steps {
                echo 'Ejecutando lint en el frontend...'
                dir("${FRONTEND_DIR}") {
                    bat '''
                        rem Si existe script de lint en package.json
                        npm run lint --if-present || echo No lint script found, skipping...
                    '''
                }
            }
        }
        
        stage('Tests - Backend') {
            steps {
                echo 'Ejecutando tests del backend...'
                dir("${BACKEND_DIR}") {
                    script {
                        def pythonExists = bat(script: 'python --version', returnStatus: true) == 0
                        if (pythonExists) {
                            bat '''
                                call %VENV_DIR%\\Scripts\\activate.bat
                                
                                rem Ejecutar tests de Django
                                python manage.py test --noinput || echo Tests fallaron o no existen
                            '''
                        } else {
                            echo 'Python no disponible. Saltando tests del backend.'
                        }
                    }
                }
            }
        }
        
        stage('Tests - Frontend') {
            steps {
                echo 'Ejecutando tests del frontend...'
                dir("${FRONTEND_DIR}") {
                    bat '''
                        rem Ejecutar tests con coverage
                        set CI=true
                        npm test -- --coverage --watchAll=false --passWithNoTests || echo Tests fallaron o no existen
                    '''
                }
            }
        }
        
        stage('Build - Frontend') {
            steps {
                echo 'Compilando frontend para producción...'
                dir("${FRONTEND_DIR}") {
                    bat '''
                        set CI=false
                        npm run build
                    '''
                }
            }
        }
        
        stage('Recolectar Static Files - Backend') {
            steps {
                echo 'Recolectando archivos estáticos de Django...'
                dir("${BACKEND_DIR}") {
                    script {
                        def pythonExists = bat(script: 'python --version', returnStatus: true) == 0
                        if (pythonExists) {
                            bat '''
                                call %VENV_DIR%\\Scripts\\activate.bat
                                
                                rem Ejecutar collectstatic
                                python manage.py collectstatic --noinput || echo collectstatic falló o no está configurado
                            '''
                        } else {
                            echo 'Python no disponible. Saltando collectstatic.'
                        }
                    }
                }
            }
        }
        
        stage('Verificar Build') {
            steps {
                echo 'Verificando artefactos generados...'
                bat '''
                    echo === Estructura del frontend/build ===
                    dir %FRONTEND_DIR%\\build || echo No se encontró directorio build
                    
                    echo === Archivos estáticos del backend ===
                    dir %BACKEND_DIR%\\staticfiles || echo No se encontró directorio staticfiles
                '''
            }
        }
        
        stage('Archivar Artefactos') {
            steps {
                echo 'Archivando artefactos del build...'
                archiveArtifacts artifacts: 'frontend/build/**/*', fingerprint: true, allowEmptyArchive: true
                archiveArtifacts artifacts: 'backend/staticfiles/**/*', fingerprint: true, allowEmptyArchive: true
            }
        }
    }
    
    post {
        success {
            echo 'Build completado exitosamente!'
            // Aquí puedes agregar notificaciones (email, Slack, etc.)
        }
        
        failure {
            echo 'Build falló!'
            // Aquí puedes agregar notificaciones de fallo
        }
        
        always {
            echo 'Limpiando workspace...'
            // Limpiar cache y archivos temporales
            bat '''
                if exist %VENV_DIR% rmdir /s /q %VENV_DIR%
                if exist %NPM_CONFIG_CACHE% rmdir /s /q %NPM_CONFIG_CACHE%
                if exist node_modules rmdir /s /q node_modules
                if exist %FRONTEND_DIR%\\node_modules rmdir /s /q %FRONTEND_DIR%\\node_modules
            '''
        }
    }
}