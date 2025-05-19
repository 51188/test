Отлично! Это довольно комплексная задача. Я предоставлю скелет проекта и основные компоненты, чтобы показать, как это может быть структурировано. Полная реализация "под ключ" займет значительно больше времени.

**Основные компоненты системы:**

1.  **Backend (Django):**
    *   API для управления сотрудниками, отделами, шаблонами.
    *   Логика заполнения шаблонов (например, `.docx` шаблонов).
    *   Отправка email с заполненным пропуском.
    *   Аутентификация (опционально, но рекомендуется).
2.  **Frontend (React):**
    *   Интерфейс для загрузки шаблонов.
    *   Форма для выбора сотрудника (с автозаполнением) и ввода доп. данных.
    *   Отображение (превью) заполняемого пропуска (сложно, если шаблон сложный).
    *   Кнопки "Отправить на Email" и "Скачать/Печать".
3.  **Database (PostgreSQL):**
    *   Хранение данных о сотрудниках, отделах, путях к шаблонам.
4.  **Docker & Docker Compose:**
    *   Контейнеризация всех сервисов для простоты развертывания и разработки.

**Структура проекта (примерная):**

```
pass-generator/
├── backend/
│   ├── Dockerfile
│   ├── manage.py
│   ├── project_name/  (Django project)
│   │   ├── __init__.py
│   │   ├── asgi.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── pass_app/      (Django app)
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── migrations/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── templates/ (если нужны Django-шаблоны для админки или серверного рендеринга)
│   ├── media/         (для загруженных шаблонов)
│   └── requirements.txt
├── frontend/
│   ├── Dockerfile
│   ├── public/
│   ├── src/
│   │   ├── components/
│   │   ├── App.js
│   │   ├── index.js
│   │   └── ...
│   ├── package.json
│   └── nginx.conf (для Nginx в Docker)
├── docker-compose.yml
└── .env (для переменных окружения)
```

Давайте начнем с `docker-compose.yml` и `Dockerfile` для каждого сервиса.

---

**1. `docker-compose.yml`:**

```yaml
version: '3.8'

services:
  db:
    image: postgres:13
    container_name: pass_db
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-pass_generator_db}
      POSTGRES_USER: ${POSTGRES_USER:-user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
    ports:
      - "5432:5432" # Осторожно: не выставляйте наружу в проде без фаервола

  backend:
    build: ./backend
    container_name: pass_backend
    command: >
      sh -c "python manage.py migrate &&
             python manage.py runserver 0.0.0.0:8000"
    volumes:
      - ./backend:/app
      - media_volume:/app/media # Для сохранения загруженных шаблонов
    ports:
      - "8000:8000"
    environment:
      - POSTGRES_NAME=${POSTGRES_DB:-pass_generator_db}
      - POSTGRES_USER=${POSTGRES_USER:-user}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-password}
      - POSTGRES_HOST=db
      - DJANGO_SECRET_KEY=${DJANGO_SECRET_KEY:-yourverysecretkeypleasechange}
      - DJANGO_DEBUG=${DJANGO_DEBUG:-True}
      - EMAIL_HOST_USER=${EMAIL_HOST_USER:-} # your.email@example.com
      - EMAIL_HOST_PASSWORD=${EMAIL_HOST_PASSWORD:-} # your_email_password
      - EMAIL_HOST=${EMAIL_HOST:-smtp.example.com}
      - EMAIL_PORT=${EMAIL_PORT:-587}
      - EMAIL_USE_TLS=${EMAIL_USE_TLS:-True}
    depends_on:
      - db
    env_file:
      - .env # Для чувствительных данных

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: pass_frontend
    ports:
      - "3000:80" # Nginx будет слушать на 80 внутри контейнера
    depends_on:
      - backend

volumes:
  postgres_data:
  media_volume:
```

**2. `.env` (создайте этот файл рядом с `docker-compose.yml`):**

```env
# PostgreSQL
POSTGRES_DB=pass_generator_db
POSTGRES_USER=user
POSTGRES_PASSWORD=supersecretpassword

# Django
DJANGO_SECRET_KEY=yourverysecretkeypleasechangeitnowforsafety
DJANGO_DEBUG=True

# Email (пример для Gmail, может потребоваться "Пароль приложения")
EMAIL_HOST_USER=your.email@gmail.com
EMAIL_HOST_PASSWORD=your_gmail_app_password
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
```

---

**3. Backend (Django)**

**`backend/Dockerfile`:**

```dockerfile
FROM python:3.9-slim

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    # Для python-docx-template (lxml)
    libxml2-dev libxslt1-dev \
    # Для psycopg2
    libpq-dev gcc \
    # Для возможной конвертации в PDF (если решите использовать unoconv/libreoffice)
    # libreoffice-writer \
    # unoconv \
    && apt-get clean && rm -rf /var/lib/apt/lists/*


COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Создаем директорию media, если ее нет
RUN mkdir -p /app/media

# Выставляем порт, который будет слушать Gunicorn/Django dev server
EXPOSE 8000

# При запуске можно использовать Gunicorn для production:
# CMD ["gunicorn", "project_name.wsgi:application", "--bind", "0.0.0.0:8000"]
# Для разработки оставим runserver
# CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
# Команда перенесена в docker-compose.yml для удобства миграций
```

**`backend/requirements.txt`:**

```txt
Django>=3.2,<4.0
djangorestframework>=3.12
psycopg2-binary>=2.8
django-cors-headers>=3.7
python-docx-template>=0.8.0 # Для работы с .docx шаблонами
# Pillow # Если будете работать с изображениями в пропусках
# reportlab # Если решите генерировать PDF с нуля
# weasyprint # Еще одна опция для HTML -> PDF
```

**`backend/project_name/settings.py` (ключевые изменения):**

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY', 'fallback-secret-key-for-dev')
DEBUG = os.environ.get('DJANGO_DEBUG', 'False').lower() in ('true', '1', 't')

ALLOWED_HOSTS = ['*'] # Для Docker и разработки. В проде укажите конкретные хосты.

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'corsheaders',
    'pass_app', # Наше приложение
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware', # Важно для React
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'project_name.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'project_name.wsgi.application'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('POSTGRES_NAME'),
        'USER': os.environ.get('POSTGRES_USER'),
        'PASSWORD': os.environ.get('POSTGRES_PASSWORD'),
        'HOST': os.environ.get('POSTGRES_HOST'),
        'PORT': '5432',
    }
}

# ... (AUTH_PASSWORD_VALIDATORS, LANGUAGE_CODE, TIME_ZONE, etc. остаются)

STATIC_URL = '/static/'
# STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles') # Для collectstatic в проде

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')


DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# CORS Settings
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000", # React dev server
    "http://127.0.0.1:3000",
    # Добавьте ваш домен в проде
]
CORS_ALLOW_CREDENTIALS = True # Если нужны cookies/auth headers

# Email settings (из .env)
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = os.environ.get('EMAIL_HOST')
EMAIL_PORT = int(os.environ.get('EMAIL_PORT', 587))
EMAIL_USE_TLS = os.environ.get('EMAIL_USE_TLS', 'True').lower() in ('true', '1', 't')
EMAIL_HOST_USER = os.environ.get('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = os.environ.get('EMAIL_HOST_PASSWORD')
DEFAULT_FROM_EMAIL = EMAIL_HOST_USER
```

**`backend/project_name/urls.py`:**

```python
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('pass_app.urls')), # API нашего приложения
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

**`backend/pass_app/models.py`:**

```python
from django.db import models
import uuid

class Department(models.Model):
    name = models.CharField(max_length=100, unique=True)

    def __str__(self):
        return self.name

class Employee(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    patronymic = models.CharField(max_length=100, blank=True, null=True)
    department = models.ForeignKey(Department, on_delete=models.SET_NULL, null=True, blank=True)
    position = models.CharField(max_length=100)
    email = models.EmailField(blank=True, null=True)
    # photo = models.ImageField(upload_to='employee_photos/', blank=True, null=True) # Опционально

    def __str__(self):
        return f"{self.last_name} {self.first_name}"

class PassTemplate(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    name = models.CharField(max_length=255)
    # Шаблон должен быть .docx файлом, содержащим плейсхолдеры типа {{ name }}, {{ department }}
    template_file = models.FileField(upload_to='pass_templates/')
    # JSON поле для описания полей, которые есть в шаблоне (для UI)
    # Например: {"name": "ФИО", "department": "Подразделение", "position": "Должность"}
    fields_description = models.JSONField(default=dict, blank=True,
                                          help_text="Описание полей в формате {'template_var': 'UI Label'}")
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name

    def get_placeholders(self):
        # Примерная функция для извлечения плейсхолдеров из DOCX
        # Реальная реализация потребует анализа файла, но python-docx-template
        # сам их найдет, если они в формате {{ placeholder }}
        # Здесь мы можем просто вернуть ключи из fields_description, если они заполнены
        # или попытаться извлечь их из файла при загрузке (более сложная задача)
        return list(self.fields_description.keys()) if self.fields_description else []
```

**`backend/pass_app/admin.py`:**

```python
from django.contrib import admin
from .models import Department, Employee, PassTemplate

admin.site.register(Department)
admin.site.register(Employee)
admin.site.register(PassTemplate)
```

**`backend/pass_app/serializers.py`:**

```python
from rest_framework import serializers
from .models import Department, Employee, PassTemplate

class DepartmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Department
        fields = '__all__'

class EmployeeSerializer(serializers.ModelSerializer):
    department_name = serializers.CharField(source='department.name', read_only=True)

    class Meta:
        model = Employee
        fields = ['id', 'first_name', 'last_name', 'patronymic', 'department', 'department_name', 'position', 'email']

class PassTemplateSerializer(serializers.ModelSerializer):
    placeholders = serializers.SerializerMethodField()

    class Meta:
        model = PassTemplate
        fields = ['id', 'name', 'template_file', 'fields_description', 'placeholders', 'created_at']
        read_only_fields = ['placeholders', 'created_at']

    def get_placeholders(self, obj):
        return obj.get_placeholders()
```

**`backend/pass_app/views.py`:**
(Это самая сложная часть. Я дам основу, но её нужно будет дорабатывать)

```python
from rest_framework import viewsets, views, status
from rest_framework.response import Response
from rest_framework.decorators import action
from rest_framework.parsers import MultiPartParser, FormParser
from django.http import HttpResponse
from django.core.mail import EmailMessage
from .models import Department, Employee, PassTemplate
from .serializers import DepartmentSerializer, EmployeeSerializer, PassTemplateSerializer
from docx_template import DocxTemplate
import io
import os
from django.conf import settings

class DepartmentViewSet(viewsets.ModelViewSet):
    queryset = Department.objects.all()
    serializer_class = DepartmentSerializer

class EmployeeViewSet(viewsets.ModelViewSet):
    queryset = Employee.objects.all().select_related('department')
    serializer_class = EmployeeSerializer

    # Для автозаполнения
    @action(detail=False, methods=['get'])
    def search(self, request):
        query = request.query_params.get('q', '')
        if query:
            # Ищем по ФИО, можно добавить и другие поля
            employees = Employee.objects.filter(
                models.Q(last_name__icontains=query) |
                models.Q(first_name__icontains=query) |
                models.Q(patronymic__icontains=query)
            ).select_related('department')[:10] # Ограничиваем количество результатов
            serializer = self.get_serializer(employees, many=True)
            return Response(serializer.data)
        return Response([])


class PassTemplateViewSet(viewsets.ModelViewSet):
    queryset = PassTemplate.objects.all()
    serializer_class = PassTemplateSerializer
    parser_classes = (MultiPartParser, FormParser) # Для загрузки файлов

    # @action(detail=True, methods=['post'])
    # def fill_and_generate(self, request, pk=None):
    #     template_obj = self.get_object()
    #     data_to_fill = request.data.get('context_data', {}) # Ожидаем JSON с данными для заполнения

    #     if not os.path.exists(template_obj.template_file.path):
    #         return Response({'error': 'Template file not found.'}, status=status.HTTP_404_NOT_FOUND)

    #     try:
    #         doc = DocxTemplate(template_obj.template_file.path)
    #         context = data_to_fill # Пример: {'name': 'Иван Иванов', 'department': 'IT'}
    #         doc.render(context)

    #         # Сохраняем во временный буфер
    #         file_stream = io.BytesIO()
    #         doc.save(file_stream)
    #         file_stream.seek(0)

    #         response = HttpResponse(file_stream, content_type='application/vnd.openxmlformats-officedocument.wordprocessingml.document')
    #         response['Content-Disposition'] = f'attachment; filename="filled_pass_{template_obj.name}.docx"'
    #         return response

    #     except Exception as e:
    #         return Response({'error': str(e)}, status=status.HTTP_500_INTERNAL_SERVER_ERROR)


class GeneratePassView(views.APIView):
    def post(self, request, *args, **kwargs):
        template_id = request.data.get('template_id')
        employee_id = request.data.get('employee_id')
        custom_data = request.data.get('custom_data', {}) # Дополнительные данные, если нужны

        try:
            template_obj = PassTemplate.objects.get(id=template_id)
            employee_obj = Employee.objects.get(id=employee_id)
        except (PassTemplate.DoesNotExist, Employee.DoesNotExist):
            return Response({'error': 'Template or Employee not found.'}, status=status.HTTP_404_NOT_FOUND)

        if not os.path.exists(template_obj.template_file.path):
            return Response({'error': 'Template file not found on server.'}, status=status.HTTP_404_NOT_FOUND)

        context = {
            'name': f"{employee_obj.last_name} {employee_obj.first_name} {employee_obj.patronymic or ''}".strip(),
            'first_name': employee_obj.first_name,
            'last_name': employee_obj.last_name,
            'patronymic': employee_obj.patronymic or '',
            'department': employee_obj.department.name if employee_obj.department else 'N/A',
            'position': employee_obj.position,
            'email': employee_obj.email or '',
            # Добавьте другие поля из Employee или custom_data, которые есть в шаблоне
            # Например, если в шаблоне есть {{ registration_number }}
            # 'registration_number': custom_data.get('registration_number', 'Не указан')
        }
        # Добавляем любые кастомные данные поверх данных сотрудника
        context.update(custom_data)


        try:
            doc = DocxTemplate(template_obj.template_file.path)
            doc.render(context) # Заполняем шаблон

            file_stream = io.BytesIO()
            doc.save(file_stream)
            file_stream.seek(0)

            action_type = request.data.get('action', 'download') # 'download' or 'email'

            if action_type == 'email':
                recipient_email = request.data.get('recipient_email', employee_obj.email)
                if not recipient_email:
                    return Response({'error': 'Recipient email not provided and employee has no email.'}, status=status.HTTP_400_BAD_REQUEST)

                email_subject = f"Ваш пропуск: {template_obj.name}"
                email_body = f"Здравствуйте, {employee_obj.first_name}!\n\nВо вложении ваш заполненный пропуск.\n\nС уважением,\nСистема Генерации Пропусков"

                email = EmailMessage(
                    email_subject,
                    email_body,
                    settings.DEFAULT_FROM_EMAIL,
                    [recipient_email]
                )
                email.attach(f"pass_{employee_obj.last_name}.docx", file_stream.getvalue(), 'application/vnd.openxmlformats-officedocument.wordprocessingml.document')
                email.send()
                return Response({'message': f'Pass sent to {recipient_email}'}, status=status.HTTP_200_OK)

            elif action_type == 'download':
                response = HttpResponse(file_stream, content_type='application/vnd.openxmlformats-officedocument.wordprocessingml.document')
                response['Content-Disposition'] = f'attachment; filename="pass_{employee_obj.last_name}_{template_obj.name}.docx"'
                return response
            else:
                return Response({'error': 'Invalid action type.'}, status=status.HTTP_400_BAD_REQUEST)

        except Exception as e:
            # Логирование ошибки здесь будет полезно
            print(f"Error generating pass: {e}")
            return Response({'error': f'Error during document generation: {str(e)}'}, status=status.HTTP_500_INTERNAL_SERVER_ERROR)
```

**`backend/pass_app/urls.py`:**

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import DepartmentViewSet, EmployeeViewSet, PassTemplateViewSet, GeneratePassView

router = DefaultRouter()
router.register(r'departments', DepartmentViewSet)
router.register(r'employees', EmployeeViewSet)
router.register(r'templates', PassTemplateViewSet)

urlpatterns = [
    path('', include(router.urls)),
    path('generate-pass/', GeneratePassView.as_view(), name='generate-pass'),
]
```

**Не забудьте создать приложение и сделать миграции:**

1.  В директории `backend/` выполните:
    ```bash
    docker-compose exec backend python manage.py startapp pass_app
    ```
    (Если вы уже создали структуру файлов, этот шаг не нужен).
2.  Добавьте `'pass_app'` в `INSTALLED_APPS` в `settings.py`.
3.  Выполните миграции (docker-compose сделает это при запуске, если команда в `command` правильная, но можно и вручную):
    ```bash
    docker-compose exec backend python manage.py makemigrations pass_app
    docker-compose exec backend python manage.py migrate
    ```
4.  Создайте суперпользователя для доступа к админке:
    ```bash
    docker-compose exec backend python manage.py createsuperuser
    ```

---

**4. Frontend (React)**

**`frontend/Dockerfile`:**

```dockerfile
# Stage 1: Build React App
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:1.21-alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**`frontend/nginx.conf` (пример, для проксирования API запросов):**

```nginx
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Проксируем API запросы к Django бэкенду
    # React будет делать запросы к /api/... на том же хосте,
    # Nginx перенаправит их на Django сервис
    location /api {
        proxy_pass http://backend:8000; # 'backend' - имя сервиса из docker-compose
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    # Проксируем media файлы (загруженные шаблоны)
    location /media {
        proxy_pass http://backend:8000;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    # Можно добавить обработку static файлов Django, если они не собираются в React
    # location /static {
    #     proxy_pass http://backend:8000;
    #     ...
    # }

    # Для WebSocket, если понадобится
    # location /ws {
    #     proxy_pass http://backend:8000;
    #     proxy_http_version 1.1;
    #     proxy_set_header Upgrade $http_upgrade;
    #     proxy_set_header Connection "Upgrade";
    # }
}
```

**`frontend/package.json` (минимальный):**

```json
{
  "name": "pass-frontend",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "@testing-library/jest-dom": "^5.16.5",
    "@testing-library/react": "^13.4.0",
    "@testing-library/user-event": "^13.5.0",
    "axios": "^1.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-scripts": "5.0.1",
    "react-select": "^5.4.0", // Для удобного автозаполнения
    "web-vitals": "^2.1.4"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {
    "extends": [
      "react-app",
      "react-app/jest"
    ]
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  },
  "proxy": "http://localhost:8000" // Для разработки без Nginx (если запускать React отдельно)
}
```
**Примечание по `"proxy"` в `package.json`**: Это для `npm start`. Когда вы билдите для Docker с Nginx, Nginx будет обрабатывать проксирование.

**Создайте React приложение (если еще нет):**
Внутри директории `frontend/`:
```bash
npx create-react-app . # Если папка пустая
# или
# npx create-react-app my-app
# cd my-app
# # скопируйте package.json и другие файлы в корень frontend/
```
Затем установите `axios` и `react-select`:
```bash
npm install axios react-select
```

**`frontend/src/App.js` (очень упрощенный пример):**

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import Select from 'react-select/async'; // Async-Select для автозаполнения
import './App.css';

// Настройте baseURL для axios
const apiClient = axios.create({
  baseURL: '/api', // Nginx будет проксировать это на http://backend:8000/api
});

function App() {
  const [templates, setTemplates] = useState([]);
  const [selectedTemplate, setSelectedTemplate] = useState(null);
  const [selectedEmployee, setSelectedEmployee] = useState(null);
  const [customFields, setCustomFields] = useState({});
  const [emailTo, setEmailTo] = useState('');
  const [message, setMessage] = useState('');

  const [newTemplateFile, setNewTemplateFile] = useState(null);
  const [newTemplateName, setNewTemplateName] = useState('');
  const [newTemplateFieldsDesc, setNewTemplateFieldsDesc] = useState('');


  useEffect(() => {
    fetchTemplates();
  }, []);

  const fetchTemplates = async () => {
    try {
      const response = await apiClient.get('/templates/');
      setTemplates(response.data);
    } catch (error) {
      console.error("Error fetching templates:", error);
      setMessage("Error fetching templates.");
    }
  };

  const loadEmployeeOptions = async (inputValue) => {
    if (!inputValue || inputValue.length < 2) {
      return [];
    }
    try {
      const response = await apiClient.get(`/employees/search/?q=${inputValue}`);
      return response.data.map(emp => ({
        value: emp.id,
        label: `${emp.last_name} ${emp.first_name} (${emp.position}, ${emp.department_name || 'N/A'})`,
        employeeData: emp // Сохраняем полные данные сотрудника
      }));
    } catch (error) {
      console.error("Error searching employees:", error);
      return [];
    }
  };

  const handleTemplateChange = (event) => {
    const templateId = event.target.value;
    const template = templates.find(t => t.id === templateId);
    setSelectedTemplate(template);
    setCustomFields({}); // Сброс кастомных полей при смене шаблона
    if (template && template.fields_description) {
        const initialCustomFields = {};
        Object.keys(template.fields_description).forEach(key => {
            initialCustomFields[key] = '';
        });
        setCustomFields(initialCustomFields);
    }
  };

  const handleCustomFieldChange = (fieldName, value) => {
    setCustomFields(prev => ({ ...prev, [fieldName]: value }));
  };

  const handleEmployeeChange = (selectedOption) => {
    setSelectedEmployee(selectedOption);
    if (selectedOption && selectedOption.employeeData) {
        setEmailTo(selectedOption.employeeData.email || '');
    } else {
        setEmailTo('');
    }
  };

  const handleGenerate = async (actionType) => {
    if (!selectedTemplate || !selectedEmployee) {
      setMessage("Please select a template and an employee.");
      return;
    }
    setMessage("Generating...");

    const payload = {
      template_id: selectedTemplate.id,
      employee_id: selectedEmployee.value,
      custom_data: customFields, // Отправляем кастомные поля
      action: actionType,
    };

    if (actionType === 'email') {
      payload.recipient_email = emailTo;
      if (!emailTo) {
        setMessage("Please enter recipient email.");
        return;
      }
    }

    try {
      const response = await apiClient.post('/generate-pass/', payload, {
        responseType: actionType === 'download' ? 'blob' : 'json', // blob для скачивания файла
      });

      if (actionType === 'download') {
        const blob = new Blob([response.data], { type: response.headers['content-type'] });
        const url = window.URL.createObjectURL(blob);
        const link = document.createElement('a');
        link.href = url;
        const contentDisposition = response.headers['content-disposition'];
        let fileName = `pass_${selectedEmployee.label.split(' ')[0]}.docx`;
        if (contentDisposition) {
            const fileNameMatch = contentDisposition.match(/filename="?(.+)"?/);
            if (fileNameMatch.length === 2)
                fileName = fileNameMatch[1];
        }
        link.setAttribute('download', fileName);
        document.body.appendChild(link);
        link.click();
        link.remove();
        window.URL.revokeObjectURL(url);
        setMessage("Pass downloaded successfully.");
      } else if (actionType === 'email') {
        setMessage(response.data.message || "Pass sent successfully.");
      }
    } catch (error) {
      console.error("Error generating pass:", error.response ? error.response.data : error);
      setMessage(`Error: ${error.response?.data?.error || 'Could not generate pass.'}`);
    }
  };

  const handleTemplateUpload = async (e) => {
    e.preventDefault();
    if (!newTemplateFile || !newTemplateName) {
        setMessage("Please provide template name and file.");
        return;
    }
    const formData = new FormData();
    formData.append('name', newTemplateName);
    formData.append('template_file', newTemplateFile);
    try {
        // Пытаемся парсить JSON из строки
        const parsedFieldsDesc = newTemplateFieldsDesc ? JSON.parse(newTemplateFieldsDesc) : {};
        formData.append('fields_description', JSON.stringify(parsedFieldsDesc));
    } catch (jsonError) {
        setMessage("Invalid JSON for fields description. Example: {\"key1\": \"Label 1\", \"key2\": \"Label 2\"}");
        return;
    }


    setMessage("Uploading template...");
    try {
        await apiClient.post('/templates/', formData, {
            headers: {
                'Content-Type': 'multipart/form-data',
            },
        });
        setMessage("Template uploaded successfully!");
        setNewTemplateName('');
        setNewTemplateFile(null);
        setNewTemplateFieldsDesc('');
        document.getElementById('template-file-input').value = null; // Сброс input file
        fetchTemplates(); // Обновить список шаблонов
    } catch (error) {
        console.error("Error uploading template:", error.response ? error.response.data : error);
        setMessage(`Error uploading template: ${error.response?.data?.detail || error.response?.data?.template_file || 'Unknown error'}`);
    }
  };


  return (
    <div className="App">
      <h1>Pass Generator</h1>
      {message && <p className={`message ${message.startsWith("Error") ? 'error' : 'success'}`}>{message}</p>}

      <div className="section">
        <h2>Upload New Template (.docx)</h2>
        <form onSubmit={handleTemplateUpload}>
          <div>
            <label htmlFor="templateName">Template Name:</label>
            <input
              type="text"
              id="templateName"
              value={newTemplateName}
              onChange={(e) => setNewTemplateName(e.target.value)}
              required
            />
          </div>
          <div>
            <label htmlFor="templateFile">Template File (.docx):</label>
            <input
              type="file"
              id="template-file-input"
              accept=".docx"
              onChange={(e) => setNewTemplateFile(e.target.files[0])}
              required
            />
          </div>
          <div>
            <label htmlFor="templateFieldsDesc">Fields Description (JSON, optional):</label>
            <textarea
              id="templateFieldsDesc"
              placeholder='e.g., {"event_name": "Event Name", "pass_id": "Pass ID"}'
              value={newTemplateFieldsDesc}
              onChange={(e) => setNewTemplateFieldsDesc(e.target.value)}
              rows="3"
            />
            <small>Example: <code>{`{"event_name": "Название события", "guest_category": "Категория гостя"}`}</code>. Эти поля будут доступны для ввода ниже.</small>
          </div>
          <button type="submit">Upload Template</button>
        </form>
      </div>

      <div className="section">
        <h2>Generate Pass</h2>
        <div>
          <label htmlFor="template-select">Select Template:</label>
          <select id="template-select" onChange={handleTemplateChange} defaultValue="">
            <option value="" disabled>-- Select a Template --</option>
            {templates.map(t => (
              <option key={t.id} value={t.id}>{t.name}</option>
            ))}
          </select>
        </div>

        {selectedTemplate && (
          <>
            <div>
              <label htmlFor="employee-select">Select Employee (type to search):</label>
              <Select
                id="employee-select"
                cacheOptions
                defaultOptions
                loadOptions={loadEmployeeOptions}
                onChange={handleEmployeeChange}
                value={selectedEmployee}
                placeholder="Type employee name..."
                isClearable
              />
            </div>

            {selectedEmployee && selectedTemplate.fields_description && Object.keys(selectedTemplate.fields_description).length > 0 && (
                <div className="custom-fields">
                    <h3>Additional Template Fields:</h3>
                    {Object.entries(selectedTemplate.fields_description).map(([key, label]) => (
                        <div key={key}>
                            <label htmlFor={`custom-${key}`}>{label || key}:</label>
                            <input
                                type="text"
                                id={`custom-${key}`}
                                value={customFields[key] || ''}
                                onChange={(e) => handleCustomFieldChange(key, e.target.value)}
                            />
                        </div>
                    ))}
                </div>
            )}


            {selectedEmployee && (
              <>
                <div className="actions">
                  <button onClick={() => handleGenerate('download')}>Download Pass (.docx)</button>
                </div>
                <div className="actions email-action">
                  <input
                    type="email"
                    placeholder="Recipient Email"
                    value={emailTo}
                    onChange={(e) => setEmailTo(e.target.value)}
                  />
                  <button onClick={() => handleGenerate('email')}>Send to Email</button>
                </div>
              </>
            )}
          </>
        )}
      </div>
       <p className="note">
        Note: Placeholders in your DOCX template should match the keys from "Fields Description"
        (e.g., <code>{`{{event_name}}`}</code>) or standard employee fields
        (<code>{`{{name}}`}</code>, <code>{`{{first_name}}`}</code>, <code>{`{{last_name}}`}</code>, <code>{`{{patronymic}}`}</code>, <code>{`{{department}}`}</code>, <code>{`{{position}}`}</code>, <code>{`{{email}}`}</code>).
      </p>
    </div>
  );
}

export default App;
```

**`frontend/src/App.css` (базовые стили):**

```css
body {
  font-family: sans-serif;
  margin: 0;
  padding: 20px;
  background-color: #f4f4f4;
  color: #333;
}

.App {
  max-width: 800px;
  margin: 20px auto;
  padding: 20px;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

h1, h2 {
  color: #333;
  text-align: center;
}
h2 {
  margin-top: 30px;
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
}

.section {
  margin-bottom: 30px;
  padding: 15px;
  border: 1px solid #ddd;
  border-radius: 5px;
}

label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

input[type="text"],
input[type="email"],
input[type="file"],
textarea,
select {
  width: calc(100% - 22px);
  padding: 10px;
  margin-bottom: 15px;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
}
textarea {
    resize: vertical;
}

button {
  background-color: #007bff;
  color: white;
  padding: 10px 15px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  margin-right: 10px;
}

button:hover {
  background-color: #0056b3;
}

.actions {
  margin-top: 20px;
}
.email-action {
    display: flex;
    align-items: center;
}
.email-action input[type="email"] {
    flex-grow: 1;
    margin-right: 10px;
    margin-bottom: 0; /* Убираем нижний отступ, т.к. он есть у родителя */
}


.message {
  padding: 10px;
  margin-bottom: 20px;
  border-radius: 4px;
  text-align: center;
}

.message.success {
  background-color: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.message.error {
  background-color: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.custom-fields {
    margin-top: 15px;
    padding: 10px;
    background-color: #f9f9f9;
    border: 1px dashed #ccc;
    border-radius: 4px;
}
.custom-fields h3 {
    margin-top: 0;
    font-size: 1.1em;
    text-align: left;
    border-bottom: none;
}
.note {
    font-size: 0.9em;
    color: #555;
    margin-top: 20px;
    padding: 10px;
    background-color: #eef;
    border-left: 3px solid #007bff;
}
.note code {
    background-color: #e0e0e0;
    padding: 2px 4px;
    border-radius: 3px;
}
```

---

**Запуск всего вместе:**

1.  Убедитесь, что Docker и Docker Compose установлены.
2.  Создайте все файлы и директории согласно структуре.
3.  Заполните `.env` своими данными (особенно для email).
4.  В корневой директории `pass-generator/` выполните:
    ```bash
    docker-compose up --build
    ```
    При первом запуске сборка может занять некоторое время.
5.  После запуска:
    *   Бэкенд будет доступен на `http://localhost:8000/api/`. Админка на `http://localhost:8000/admin/`.
    *   Фронтенд будет доступен на `http://localhost:3000/`.

**Что нужно будет доделать/улучшить:**

1.  **Заполнение шаблона `PassTemplate.fields_description`:** При загрузке шаблона DOCX можно попытаться автоматически извлечь плейсхолдеры (например, все, что в `{{ }}`) и предложить пользователю дать им понятные названия. Или просто обязать пользователя заполнять это поле в формате JSON.
2.  **Превью пропуска:** Это самая сложная часть.
    *   Если шаблон DOCX, то для превью его нужно конвертировать в изображение или PDF. В Docker-контейнере для этого можно использовать `libreoffice --headless --convert-to pdf` или `unoconv`. Затем PDF можно отобразить в iframe. Это добавит зависимостей в Dockerfile бэкенда.
    *   Проще всего – не делать превью, а сразу давать скачать заполненный DOCX.
3.  **Печать:** Обычно это делается на стороне клиента. Если у вас есть PDF в iframe, можно вызвать `window.print()`. Если скачали DOCX, пользователь печатает его сам.
4.  **Обработка ошибок:** Более детальная на фронтенде и бэкенде.
5.  **Аутентификация и авторизация:** Кто может загружать шаблоны, кто генерировать пропуски?
6.  **Валидация данных:** На обеих сторонах.
7.  **Тестирование.**
8.  **Оптимизация для продакшена:** Использовать Gunicorn для Django, настроить Nginx более тщательно, `DEBUG=False` и т.д.
9.  **Пользовательский интерфейс:** Сделать его более дружелюбным и красивым.
10. **Конвертация DOCX в PDF на сервере (опционально, но полезно):**
    Если вы хотите отправлять PDF по почте или для скачивания, вам понадобится инструмент конвертации. `unoconv` (требует LibreOffice) - популярный выбор.
    *   Добавьте `unoconv libreoffice-writer` в `apt-get install` в `backend/Dockerfile`.
    *   В `GeneratePassView` после `doc.save(file_stream)` можно будет вызвать `subprocess.run(['unoconv', '-f', 'pdf', '--stdout', temp_docx_path])` (сохранив DOCX во временный файл) и прочитать stdout для получения PDF.

Это большой проект, но данный скелет должен дать хороший старт. Удачи!
