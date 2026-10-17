node {
    stage('Checkout') {
        checkout scm
    }
    stage('Build') {
        sh 'docker compose up -d --build'
    }
    stage('Database schema') {
        sh 'docker exec -i todoappdb mariadb -utodo_usr -pletmeinplz todo_db < TodoApp/schema.sql'
    }
    stage('Test') {
        sh 'curl --fail --silent --show-error --retry 10 --retry-delay 3 --retry-connrefused http://172.16.0.10:8081/ | grep -i "todo"'
    }
}