pipeline {
    agent any

    environment {
        COMPOSE_PROJECT_NAME = "resource-link-hub"
        // Gunakan host.docker.internal agar curl dari dalam Jenkins container
        // bisa reach port yang ter-expose di host machine (Docker Desktop Windows/Mac)
        BACKEND_URL          = "http://host.docker.internal:5000/links"
        FRONTEND_URL         = "http://host.docker.internal:8081"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        // ── Stage 1: Checkout ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo '📥 Cloning source code from SCM...'
                checkout scm
                echo "✅ Checkout complete — Branch: ${env.GIT_BRANCH ?: 'N/A'} | Commit: ${env.GIT_COMMIT?.take(7) ?: 'N/A'}"
            }
        }

        // ── Stage 2: Syntax Check ─────────────────────────────────────────
        stage('Syntax Check') {
            steps {
                echo '🔍 Running syntax validation...'

                // Validate Python source
                sh '''
                    echo "[1/2] Python syntax check → app.py"
                    python3 -m py_compile app.py
                    echo "      ✅ app.py passed"
                '''

                // Validate Docker Compose configuration
                sh '''
                    echo "[2/2] Docker Compose config validation"
                    docker compose config -q
                    echo "      ✅ docker-compose.yml is valid"
                '''
            }
        }

        // ── Stage 3: Build Image ──────────────────────────────────────────
        stage('Build Image') {
            steps {
                echo '🐳 Building backend Docker image (no cache)...'
                sh '''
                    docker compose build --no-cache backend
                    echo "✅ Backend image built successfully"
                '''
            }
        }

        // ── Stage 4: Automated Deployment ────────────────────────────────
        stage('Automated Deployment') {
            steps {
                echo '🚀 Stopping any conflicting containers before deploy...'
                sh '''
                    # Hapus container lama dengan nama yang sama (jika ada)
                    docker rm -f resource-link-backend resource-link-frontend 2>/dev/null || true
                    echo "✅ Old containers cleared"
                '''

                echo '🚀 Deploying all services with Docker Compose...'
                sh '''
                    docker compose up -d --remove-orphans
                    echo "✅ Services deployed:"
                    docker compose ps
                '''
            }
        }

        // ── Stage 5: Health Check ─────────────────────────────────────────
        stage('Health Check') {
            steps {
                echo '🏥 Waiting 5 seconds for services to initialize...'
                sh 'sleep 5'

                echo '🔎 Checking backend API (port 5000)...'
                sh '''
                    curl -f --silent --show-error --max-time 10 ${BACKEND_URL}
                    echo "\n✅ Backend is healthy — GET /links responded OK"
                '''

                echo '🔎 Checking frontend (port 8081)...'
                sh '''
                    curl -f --silent --show-error --max-time 10 ${FRONTEND_URL}
                    echo "\n✅ Frontend is healthy — Nginx responded OK"
                '''
            }
        }
    }

    // ── Post Actions ──────────────────────────────────────────────────────
    post {
        success {
            echo '''
╔══════════════════════════════════════════════╗
║  ✅  PIPELINE SUCCEEDED                      ║
║  Resource Link Hub deployed and healthy!     ║
║  UI  → http://localhost:8081                 ║
║  API → http://localhost:5000/links           ║
╚══════════════════════════════════════════════╝
            '''
        }
        failure {
            echo '❌ Pipeline FAILED — lihat log untuk detail.'
            echo '⚠️  Container TIDAK dihapus otomatis. Cek manual: docker compose ps'
            sh 'docker compose ps || true'
        }
        always {
            echo '📋 Final container status:'
            sh 'docker compose ps || true'
        }
    }
}

