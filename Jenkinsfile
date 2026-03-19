// Build Python from source on Ubuntu LTS.
// For Jenkins-in-Docker:
// mount the host Docker socket into the Jenkins container, e.g.
//   -v /var/run/docker.sock:/var/run/docker.sock
// and ensure the Jenkins user can talk to the socket (group docker or root), or use a Docker Cloud / Kubernetes agent instead.

pipeline {
    agent {
        docker {
            // Pin to an Ubuntu LTS tag; adjust as needed (e.g. ubuntu:24.04).
            image 'ubuntu:22.04'
            // Build deps and install need root; use a dedicated image in production if you prefer non-root builds.
            args '-u root --network=bridge'
            // Optional: reuse workspace
            // reuseNode true
        }
    }

    options {
        timestamps()
        timeout(time: 2, unit: 'HOURS')
        // Keep logs from long configure/make output manageable
        ansiColor('xterm')
    }

    parameters {
        string(name: 'PYTHON_VERSION', defaultValue: '3.12.8', description: 'Full version string, e.g. 3.12.8 (see python.org source releases)')
        string(name: 'INSTALL_PREFIX', defaultValue: '/usr/local', description: 'Prefix for make altinstall')
        booleanParam(name: 'RUN_TESTS', defaultValue: false, description: 'Run "make test" after build (slow)')
    }

    environment {
        // Avoid interactive debconf during apt
        DEBIAN_FRONTEND = 'noninteractive'
    }

    stages {
        stage('Prepare') {
            steps {
                sh '''
                    set -eux
                    apt-get update
                    apt-get install -y --no-install-recommends \
                        ca-certificates \
                        curl \
                        gnupg \
                        xz-utils \
                        build-essential \
                        pkg-config \
                        libssl-dev \
                        zlib1g-dev \
                        libbz2-dev \
                        libreadline-dev \
                        libsqlite3-dev \
                        libffi-dev \
                        libncursesw5-dev \
                        libgdbm-dev \
                        libc6-dev \
                        liblzma-dev \
                        tk-dev \
                        uuid-dev
                '''
            }
        }

        stage('Fetch source') {
            steps {
                sh '''
                    set -eux
                    VER="${PYTHON_VERSION}"
                    MAJOR_MINOR="$(echo "$VER" | cut -d. -f1-2)"
                    TARBALL="Python-${VER}.tgz"
                    URL="https://www.python.org/ftp/python/${VER}/${TARBALL}"
                    rm -rf "Python-${VER}" "${TARBALL}"
                    curl -fsSL -o "${TARBALL}" "${URL}"
                    tar -xzf "${TARBALL}"
                '''
            }
        }

        stage('Configure & build') {
            steps {
                sh '''
                    set -eux
                    cd "Python-${PYTHON_VERSION}"
                    ./configure \
                        --prefix="${INSTALL_PREFIX}" \
                        --enable-optimizations \
                        --with-ensurepip=install
                    make -j"$(nproc)"
                '''
            }
        }

        stage('Test (optional)') {
            when {
                expression { return params.RUN_TESTS }
            }
            steps {
                sh '''
                    set -eux
                    cd "Python-${PYTHON_VERSION}"
                    make test
                '''
            }
        }

        stage('Install') {
            steps {
                sh '''
                    set -eux
                    cd "Python-${PYTHON_VERSION}"
                    # altinstall avoids replacing the system "python3" on distros that ship one
                    make altinstall
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    set -eux
                    MAJOR_MINOR="$(echo "${PYTHON_VERSION}" | cut -d. -f1-2)"
                    "${INSTALL_PREFIX}/bin/python${MAJOR_MINOR}" -V
                    "${INSTALL_PREFIX}/bin/python${MAJOR_MINOR}" -m pip --version
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            echo 'Build failed — check console for configure/make errors.'
        }
    }
}
