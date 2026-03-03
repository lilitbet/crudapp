pipeline {
  agent { label 'docker-agent' }
  environment {
    APP_NAME = 'app'
    CANARY_APP_NAME = 'app-canary'
    DOCKER_HUB_USER = 'lilitbet'
    GIT_REPO = 'https://github.com/lilitbet/crudapp.git'
    BACKEND_IMAGE_NAME = 'crudback'
    DATABASE_IMAGE_NAME = 'mysql'
    MANAGER_IP = '192.168.0.1'
    PHP_FILE = 'regist.php'
  }

  stages {
    stage('Checkout') {
      steps {
        git url: "${GIT_REPO}", branch: 'main'
      }
    }

    // ИСПРАВЛЕННЫЙ ЭТАП: Проверка required атрибутов в regist.php
    stage('Validate Required Attributes') {
      steps {
        script {
          echo "=== Проверка атрибута required в файле ${PHP_FILE} ==="
          
          // Проверяем существование файла
          if (!fileExists(PHP_FILE)) {
            error "Файл ${PHP_FILE} не найден!"
          }
          
          // Читаем содержимое файла
          def content = readFile(PHP_FILE)
          
          // Находим все input поля (исправленное регулярное выражение)
          def inputPattern = '<input\\b[^>]*>'
          def inputs = (content =~ /$inputPattern/i)
          
          def missingRequired = []
          def totalInputs = 0
          def skippedInputs = 0
          
          // Проверяем каждое поле
          for (input in inputs) {
            totalInputs++
            
            // Проверяем, является ли поле hidden, submit, button, reset или image
            // Используем отдельное регулярное выражение без модификатора в строке
            def skipPattern = 'type=["\']?(hidden|submit|button|reset|image)["\']?'
            def matcher = (input =~ /$skipPattern/i)
            
            if (matcher.find()) {
              skippedInputs++
              echo "Пропущено поле (${matcher[0][1]}): ${input.substring(0, Math.min(50, input.length()))}..."
              continue
            }
            
            // Проверяем наличие атрибута required
            def requiredPattern = '\\srequired\\b'
            if (!(input =~ /$requiredPattern/i)) {
              missingRequired.add(input.trim())
              echo "❌ Поле без required: ${input.substring(0, Math.min(50, input.length()))}..."
            }
          }
          
          // Выводим статистику
          echo "================================="
          echo "Всего найдено полей input: ${totalInputs}"
          echo "Пропущено (hidden/button и т.д.): ${skippedInputs}"
          echo "Проверено полей: ${totalInputs - skippedInputs}"
          echo "Поля с required: ${(totalInputs - skippedInputs) - missingRequired.size()}"
          echo "Поля без required: ${missingRequired.size()}"
          echo "================================="
          
          // Создаем отчет
          def reportFile = 'required-check-report.txt'
          def reportContent = """
=== ОТЧЕТ ПРОВЕРКИ REQUIRED ===
Файл: ${PHP_FILE}
Время: ${new Date()}
Build: ${BUILD_NUMBER}

Статистика:
- Всего полей input: ${totalInputs}
- Пропущено (hidden/button): ${skippedInputs}
- Проверено полей: ${totalInputs - skippedInputs}
- Поля с required: ${(totalInputs - skippedInputs) - missingRequired.size()}
- Поля без required: ${missingRequired.size()}
"""
          
          if (missingRequired.size() > 0) {
            reportContent += "\n\nСписок полей БЕЗ атрибута required:\n"
            missingRequired.eachWithIndex { input, index ->
              reportContent += "${index + 1}. ${input}\n"
            }
            
            // Сохраняем отчет
            writeFile file: reportFile, text: reportContent
            echo reportContent
            
            // Показываем отчет в Jenkins
            archiveArtifacts artifacts: reportFile, fingerprint: true
            
            currentBuild.result = 'UNSTABLE'
            error "Найдено ${missingRequired.size()} полей ввода без атрибута required! Отчет сохранен в ${reportFile}"
          } else {
            reportContent += "\n\n✅ ВСЕ ПОЛЯ ИМЕЮТ АТРИБУТ REQUIRED!"
            
            // Сохраняем отчет
            writeFile file: reportFile, text: reportContent
            echo reportContent
            
            echo "✓ Проверка required пройдена успешно!"
          }
        }
      }
    }

    stage('Build Docker Images') {
      steps {
        sh "docker build -f php.Dockerfile . -t ${DOCKER_HUB_USER}/${BACKEND_IMAGE_NAME}:${BUILD_NUMBER}"
        sh "docker build -f mysql.Dockerfile . -t ${DOCKER_HUB_USER}/${DATABASE_IMAGE_NAME}:${BUILD_NUMBER}"
      }
    }

    stage('Push to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
            docker push ${DOCKER_HUB_USER}/${BACKEND_IMAGE_NAME}:${BUILD_NUMBER}
            docker push ${DOCKER_HUB_USER}/${DATABASE_IMAGE_NAME}:${BUILD_NUMBER}
          '''
        }
      }
    }

    stage('Deploy Canary') {
      steps {
        sh '''
          echo "=== Развёртывание Canary (1 реплика) ==="
          docker stack deploy -c docker-compose_canary.yaml ${CANARY_APP_NAME} --with-registry-auth
          sleep 40
          docker service ls --filter name=${CANARY_APP_NAME}
        '''
      }
    }

    stage('Canary Testing') {
      steps {
        sh '''
          echo "=== Тестирование Canary-версии (порт 8081) ==="
          SUCCESS=0
          TESTS=10
          for i in $(seq 1 $TESTS); do
            echo "Тест $i/$TESTS..."
            if curl -f -s --max-time 15 http://${MANAGER_IP}:8081/ > /tmp/canary_$i.html; then
              if ! grep -iq "error\\|fatal\\|exception\\|failed" /tmp/canary_$i.html; then
                SUCCESS=$((SUCCESS + 1))
                echo "✓ Тест $i пройден"
              else
                echo "✗ Тест $i: найдены ошибки в ответе"
              fi
            else
              echo "✗ Тест $i: нет ответа"
            fi
            sleep 6
          done
          echo "Успешных тестов: $SUCCESS/$TESTS"
          [ "$SUCCESS" -ge 8 ] || exit 1
          echo "Canary прошёл тестирование!"
        '''
      }
    }

    stage('Gradual Traffic Shift') {
      steps {
        sh '''
          echo "=== Постепенное переключение трафика на новую версию ==="
          # Проверяем, существует ли основной сервис
          if docker service ls --filter name=${APP_NAME}_web-server | grep -q ${APP_NAME}_web-server; then
            echo "Основной сервис существует — начинаем rolling update по одной реплике"

            # Шаг 1: Обновляем первую реплику продакшена (33% трафика на v${BUILD_NUMBER})
            echo "Шаг 1: Обновляем 1-ю реплику продакшена"
            docker service update \
              --image ${DOCKER_HUB_USER}/${BACKEND_IMAGE_NAME}:${BUILD_NUMBER} \
              --update-parallelism 1 \
              --update-delay 20h \
              --detach=true \
              ${APP_NAME}_web-server

            echo "Ожидание стабилизации после первой реплики..."
            sleep 40
            docker service ps ${APP_NAME}_web-server --no-trunc | head -20

            # Мониторинг после первого шага
            echo "=== Мониторинг после первой реплики ==="
            MONITOR_SUCCESS=0
            MONITOR_TESTS=10
            for j in $(seq 1 $MONITOR_TESTS); do
              if curl -f -s --max-time 15 http://${MANAGER_IP}:8080/ > /tmp/monitor_$j.html; then
                if ! grep -iq "error\\|fatal" /tmp/monitor_$j.html; then
                  MONITOR_SUCCESS=$((MONITOR_SUCCESS + 1))
                fi
              fi
              sleep 5
            done
            echo "Успешных проверок после первой реплики: $MONITOR_SUCCESS/$MONITOR_TESTS"
            [ "$MONITOR_SUCCESS" -ge 8 ] || exit 1

            echo "Мониторинг после первой реплики прошёл!"
            sleep 60

            # Шаг 2: Обновляем оставшиеся реплики
            echo "Шаг 2: Обновляем оставшиеся реплики"
            docker service update \
              --image ${DOCKER_HUB_USER}/${BACKEND_IMAGE_NAME}:${BUILD_NUMBER} \
              --update-parallelism 1 \
              --update-delay 30s \
              ${APP_NAME}_web-server

            echo "Ожидание завершения полного обновления..."
            sleep 90

            # Проверяем, что все реплики обновлены
            echo "Статус после обновления:"
            docker service ps ${APP_NAME}_web-server | head -20

            # Удаляем canary — он больше не нужен
            echo "Удаление canary stack..."
            docker stack rm ${CANARY_APP_NAME} || true
            sleep 20
          else
            echo "Первый деплой — разворачиваем продакшен"
            docker stack deploy -c docker-compose.yaml ${APP_NAME} --with-registry-auth
            sleep 60
          fi

          echo "Постепенное переключение завершено"
        '''
      }
    }

    stage('Final Verification') {
      steps {
        sh '''
          echo "=== Финальная проверка ==="
          for i in $(seq 1 5); do
            echo "Финальный тест $i/5..."
            if curl -f --max-time 10 http://${MANAGER_IP}:8080/ > /dev/null 2>&1; then
              echo "✓ Тест $i пройден"
            else
              echo "✗ Тест $i не пройден"
              exit 1
            fi
            sleep 5
          done
          echo "Все финальные тесты пройдены!"
        '''
      }
    }
  }

  post {
    success {
      echo "✓ Canary-деплой успешно завершён!"
      sh 'docker logout'
    }
    failure {
      echo "✗ Ошибка в пайплайне — canary удалён, продакшен остался прежним"
      sh '''
        docker stack rm ${CANARY_APP_NAME} || true
        echo "Canary удалён, продакшен не тронут"
      '''
      sh 'docker logout'
    }
    always {
      sh 'docker image prune -f || true'
    }
  }
}
