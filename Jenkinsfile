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
    PHP_FILE = 'app/regist.php'
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
          
          // Простой способ найти все input теги
          def inputs = []
          def index = 0
          
          while (true) {
            def startTag = content.indexOf('<input', index)
            if (startTag == -1) {
              startTag = content.indexOf('<INPUT', index)
              if (startTag == -1) break
            }
            
            def endTag = content.indexOf('>', startTag)
            if (endTag == -1) break
            
            def inputTag = content.substring(startTag, endTag + 1)
            inputs.add(inputTag)
            
            index = endTag + 1
          }
          
          def missingRequired = []
          def totalInputs = inputs.size()
          def skippedInputs = 0
          
          echo "Найдено полей input: ${totalInputs}"
          
          // Проверяем каждое поле
          inputs.each { input ->
            def lowerInput = input.toLowerCase()
            
            // Пропускаем скрытые поля и кнопки
            if (lowerInput.contains('type="hidden"') || 
                lowerInput.contains('type="submit"') || 
                lowerInput.contains('type="button"') || 
                lowerInput.contains('type="reset"') || 
                lowerInput.contains('type="image"') ||
                lowerInput.contains("type='hidden'") || 
                lowerInput.contains("type='submit'") || 
                lowerInput.contains("type='button'") || 
                lowerInput.contains("type='reset'") || 
                lowerInput.contains("type='image'")) {
              skippedInputs++
              echo "Пропущено поле (type): ${input.substring(0, Math.min(50, input.length()))}..."
              return
            }
            
            // Проверяем наличие required
            if (!lowerInput.contains('required') && !lowerInput.contains('required=')) {
              missingRequired.add(input)
              echo "❌ Найдено поле без required: ${input.substring(0, Math.min(50, input.length()))}..."
            }
          }
          
          // Выводим статистику
          echo "================================="
          echo "РЕЗУЛЬТАТЫ ПРОВЕРКИ:"
          echo "================================="
          echo "Всего найдено полей input: ${totalInputs}"
          echo "Пропущено (hidden/button и т.д.): ${skippedInputs}"
          echo "Проверено полей: ${totalInputs - skippedInputs}"
          echo "Поля с required: ${(totalInputs - skippedInputs) - missingRequired.size()}"
          echo "Поля без required: ${missingRequired.size()}"
          echo "================================="
          
          // Если есть поля без required - останавливаем pipeline
          if (missingRequired.size() > 0) {
            echo "❌ ОШИБКА: Найдено ${missingRequired.size()} полей ввода без атрибута required!"
            echo "Список проблемных полей:"
            missingRequired.eachWithIndex { input, index ->
              echo "  ${index + 1}. ${input}"
            }
            error "Проверка required не пройдена! Найдены поля без атрибута required."
          } else {
            echo "✅ Проверка required пройдена успешно! Все поля имеют атрибут required."
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
