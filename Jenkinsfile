node {
    stage('Checkout') {
        checkout scm
    }
    stage('Build staging') {
        sh 'docker compose -p todoapp-staging up -d --build'
    }
    stage('Database schema staging') {
        sh 'docker exec -i todoappdb mariadb -utodo_usr -pletmeinplz todo_db < TodoApp/schema.sql'
    }
    stage('Test staging') {
        sh 'curl --fail --silent --show-error --retry 10 --retry-delay 3 --retry-connrefused http://172.16.0.10:8081/ | grep -i "todo"'
    }
    stage('Deploy production') {
        withCredentials([
            sshUserPrivateKey(credentialsId: 'dmz010-deploy', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
            string(credentialsId: 'todo-db-password', variable: 'DB_PASS')
        ]) {
            sh '''
                remote() {
                    ssh -i "$SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/var/jenkins_home/known_hosts "$SSH_USER@192.0.2.10" "$@"
                }

                # 1. Het geteste image van staging taggen en naar dmz010 sturen
                IMAGE=$(docker inspect --format '{{.Image}}' todoapp)
                docker tag "$IMAGE" todoapp:release
                docker save todoapp:release | remote docker load

                # 2. Geheimen in bestanden op dmz010 zetten (alleen leesbaar voor de deploy-gebruiker)
                { echo "ConnectionStrings__TodoDb=Server=192.0.2.20;Port=3306;Database=db1;User=user1;Password=$DB_PASS;"; echo "ASPNETCORE_ENVIRONMENT=Production"; } | remote 'umask 077; cat > todoapp.env'
                echo "MYSQL_PWD=$DB_PASS" | remote 'umask 077; cat > todoapp-db.env'

                # 3. Schema laden in db1, vanaf dmz010 (user1 mag alleen vanaf dat IP verbinden)
                remote docker run --rm -i --env-file todoapp-db.env mariadb:11 mariadb -h192.0.2.20 -uuser1 db1 < TodoApp/schema.sql

                # 4. Oude productiecontainer vervangen
                remote docker rm -f todoapp-prod || true
                remote docker run -d --restart unless-stopped --name todoapp-prod --env-file todoapp.env -p 127.0.0.1:8081:8080 todoapp:release
            '''
        }
    }
    stage('Test production') {
        sh 'curl --fail --silent --show-error --insecure --resolve www.infra.lan:8443:192.0.2.10 --retry 10 --retry-delay 3 --retry-connrefused https://www.infra.lan:8443/ | grep -i "todo"'
    }
}