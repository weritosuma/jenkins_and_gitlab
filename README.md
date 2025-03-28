---

### **Полное руководство: Сборка веток и деплой через Jenkins с GitLab**  
**Jenkins** — сервер непрерывной интеграции (Continuous Integration, CI), который автоматизирует сборку, тестирование и деплой проектов. **GitLab** — платформа для управления репозиториями и CI/CD. В этой статье разберём, как интегрировать их для автоматизации.

---

### **Часть 1: Настройка Jenkins для работы с GitLab**
#### **1. Установка Jenkins**
- **Linux**:  
  ```bash
  sudo apt update && sudo apt install jenkins
  sudo systemctl start jenkins  # Запуск службы
  ```
- **Windows/macOS**: Используйте Docker:  
  ```bash
  docker run -p 8080:8080 -p 50000:50000 jenkins/jenkins:lts
  ```
- Откройте `http://localhost:8080`, следуйте инструкциям мастера установки.

#### **2. Установка плагинов**
Перейдите в **Manage Jenkins (Управление Jenkins) → Manage Plugins (Управление плагинами)**:
- Установите плагины:
  - **GitLab Plugin** (интеграция с GitLab).
  - **Pipeline (Пайплайн)** — для создания конвейеров.
  - **Blue Ocean** — визуальный интерфейс.
  - **Docker Pipeline** (если нужен Docker).

#### **3. Настройка интеграции с GitLab**
- В GitLab:
  1. Перейдите в **Settings (Настройки) → Access Tokens (Токены доступа)**.
  2. Создайте **Personal Access Token** с правами `api` и `read_repository`.
  3. Скопируйте токен.
- В Jenkins:
  1. **Manage Jenkins → Configure System (Настройка системы)**.
  2. В секции **GitLab** нажмите **Add GitLab Server**.
  3. Укажите:
     - **Name**: `GitLab Server`.
     - **GitLab URL**: URL вашего GitLab (например, `https://gitlab.com`).
     - **Credentials**: Добавьте токен из GitLab.

---

### **Часть 2: Создание Pipeline (Пайплайн)**
#### **1. Что такое Jenkinsfile?**
**Jenkinsfile** — текстовый файл в корне репозитория, который описывает пайплайн (последовательность этапов сборки, тестирования, деплоя).  
**Правила составления**:
- Используйте синтаксис **Declarative Pipeline** (рекомендуется) или **Scripted Pipeline**.
- Каждый пайплайн состоит из **stages** (этапов), которые содержат **steps** (шаги).
- Пример структуры:
  ```groovy
  pipeline {
      agent any  // Агент (сервер) для выполнения задач
      stages {
          stage('Build') {
              steps {
                  sh 'npm install'  // Выполнить команду в shell
              }
          }
      }
  }
  ```

#### **2. Создание Pipeline-задачи**
1. В Jenkins нажмите **New Item (Создать задачу)**.
2. Введите название, выберите **Pipeline (Пайплайн)**, нажмите **OK**.
3. В настройках задачи:
   - **Pipeline** → **Definition**: `Pipeline script from SCM`.
   - **SCM**: `Git`.
   - **Repositories** → **Repository URL**: URL вашего GitLab-репозитория (например, `https://gitlab.com/your-project.git`).
   - **Branches to build**: Укажите ветку (например, `main` или `feature/*`).
   - **Script Path**: Путь к `Jenkinsfile` (по умолчанию `Jenkinsfile`).

#### **3. Пример Jenkinsfile для GitLab**
```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://gitlab.com/your-project.git'
            }
        }
        stage('Build') {
            steps {
                sh 'echo "Building..."'
                sh './gradlew build'  // Для Java-проектов
            }
        }
        stage('Test') {
            steps {
                sh './gradlew test'
            }
        }
        stage('Deploy to Staging') {
            when {
                branch 'main'  // Выполнять только для ветки main
            }
            steps {
                sshagent(['server-ssh-credentials']) {
                    sh 'scp build/libs/*.jar user@server:/opt/app'
                }
            }
        }
    }
}
```

---

### **Часть 3: Автоматизация через GitLab Webhooks**
1. В GitLab:
   - Перейдите в **Settings → Webhooks**.
   - В поле **URL** введите: `http://<jenkins-url>/gitlab/project/<project-id>`.
   - Выберите события: **Push events**, **Merge request events**.
   - Нажмите **Add webhook**.
2. В Jenkins:
   - В настройках задачи включите **Build when a change is pushed to GitLab**.

---

### **Часть 4: Деплой через Pipeline**
#### **1. Настройка SSH-доступа**
- В Jenkins:
  1. **Manage Jenkins → Manage Credentials**.
  2. Добавьте SSH-ключи для доступа к серверу (например, `server-ssh-credentials`).

#### **2. Пример этапа деплоя**
```groovy
stage('Deploy to Production') {
    steps {
        sshagent(['server-ssh-credentials']) {
            sh 'ssh user@server "docker-compose -f prod.yml up -d"'
        }
    }
}
```

---

### **Часть 5: Open Blue Ocean**
**Open Blue Ocean** — визуальный интерфейс Jenkins для работы с пайплайнами.  
**Как использовать**:
1. Перейдите в **Blue Ocean** через главную страницу Jenkins.
2. Выберите **Create Pipeline** → **GitLab** → авторизуйтесь.
3. Выберите репозиторий и ветку. Blue Ocean автоматически предложит создать `Jenkinsfile`, если его нет.

---

### **Часть 6: Лучшие практики для Pipeline**
1. **Модульность**: Разделяйте пайплайны на секции (`stages`).
2. **Параметризация**: Используйте `parameters` для настройки сборки.
   ```groovy
   pipeline {
       agent any
       parameters {
           string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
       }
       stages {
           stage('Checkout') {
               steps {
                   git branch: params.BRANCH, url: 'https://gitlab.com/your-project.git'
               }
           }
       }
   }
   ```
3. **Кэширование**: Используйте `cache` для ускорения сборки (например, для зависимостей Node.js).

---

### **Советы**
- Для отладки пайплайна используйте `echo "Debug message"`.
- Логи сборки доступны в **Console Output** (в деталях задачи).
- Используйте **Docker agents** для изоляции окружений:
  ```groovy
  agent {
      docker { image 'node:18' }
  }
  ```

Если возникнут ошибки, уточните их текст — помогу разобраться!
