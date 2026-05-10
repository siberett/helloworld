pipeline {
    agent {
        docker {
            image 'local/ci-runner:latest'
            args '--network jenkins_ci_net'
        }
    }

    stages {
        stage('Get Code') {
            steps {
                cleanWs()

                git 'https://github.com/siberett/helloworld.git'
            }
        }

        stage('Rest') {
            steps {
                sh '''
                    set +e
        
                    export PYTHONPATH="$WORKSPACE"
        
                    flask --app app.api:api_application run --host 127.0.0.1 --port 5001 > flask.log 2>&1 &
                    FLASK_PID=$!
        
                    trap "kill $FLASK_PID 2>/dev/null || true" EXIT
        
                    for i in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1:5001/calc/add/1/2 >/dev/null; then
                            break
                        fi
                        sleep 1
                    done
        
                    curl -fsS http://wiremock:8080/calc/sqrt/64
        
                    APP_BASE_URL=http://127.0.0.1:5001 \\
                    WIREMOCK_BASE_URL=http://wiremock:8080 \\
                    pytest --junitxml=result-rest.xml test/rest
        
                    exit 0
                '''
            }
        }
        
        stage('Coverage') {
            steps {
                sh '''
                    export PYTHONPATH="$WORKSPACE"
        
                    coverage run --branch --source=app --omit=app/__init__.py,app/api.py -m pytest --junitxml=result-unit.xml test/unit || true
                    coverage xml
                '''
        
                junit 'result-unit.xml'
        
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    recordCoverage(
                        tools: [[parser: 'COBERTURA', pattern: 'coverage.xml']],
                        qualityGates: [
                            [threshold: 85.0, metric: 'LINE', baseline: 'PROJECT', unstable: false],
                            [threshold: 95.0, metric: 'LINE', baseline: 'PROJECT', unstable: true],
        
                            [threshold: 80.0, metric: 'BRANCH', baseline: 'PROJECT', unstable: false],
                            [threshold: 90.0, metric: 'BRANCH', baseline: 'PROJECT', unstable: true]
                        ],
                        sourceCodeRetention: 'LAST_BUILD'
                    )
                }
            }
        }        
        
        stage('Static') {
            steps {
                sh '''
                    flake8 --exit-zero --format=pylint app > flake8.out
                '''
    
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    recordIssues(
                        tools: [flake8(name: 'Flake8', pattern: 'flake8.out')],
                        qualityGates: [
                            [threshold: 10, type: 'TOTAL', unstable: false],
                            [threshold: 8, type: 'TOTAL', unstable: true]
                        ]
                    )
                }
            }
        }

        stage('Security') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh '''
                        bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                    '''
                }

                recordIssues(
                    tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')],
                    qualityGates: [
                        [threshold: 4, type: 'TOTAL', unstable: false],
                        [threshold: 2, type: 'TOTAL', unstable: true]
                    ]
                )
            }
        }

        stage('Performance') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh '''
                        set -e

                        export PYTHONPATH="$WORKSPACE"

                        flask --app app.api:api_application run --host 127.0.0.1 --port 5001 > flask-performance.log 2>&1 &
                        FLASK_PID=$!

                        trap "kill $FLASK_PID 2>/dev/null || true" EXIT

                        for i in $(seq 1 30); do
                            if curl -fsS http://127.0.0.1:5001/calc/add/1/2 >/dev/null; then
                                break
                            fi
                            sleep 1
                        done

                        jmeter -n -t test/jmeter/flask.jmx -f -l flask.jtl
                    '''
                }

                perfReport sourceDataFiles: 'flask.jtl'
            }
        }
    }
}
