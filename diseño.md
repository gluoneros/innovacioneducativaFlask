# Innovacioneducativa - Sistema de Gestión Escolar Backend
## Documento de Diseño Técnico y Guía de Implementación

## 1. Arquitectura de Software

### 1.1 Estructura de Carpetas del Proyecto


```
innovacioneducativa/
├── app/
│   ├── __init__.py                 # Application Factory
│   ├── extensions.py               # Inicialización de extensiones (db, jwt, migrate, etc.)
│   ├── config.py                   # Configuraciones por entorno
│   │
│   ├── models/                     # Modelos SQLAlchemy
│   │   ├── __init__.py
│   │   ├── user.py                 # Modelo User
│   │   ├── course.py               # Modelo Course
│   │   ├── grade.py                # Modelo Grade
│   │   └── guardian_student.py     # Tabla intermedia Acudiente-Estudiante
│   │
│   ├── schemas/                    # Schemas Marshmallow
│   │   ├── __init__.py
│   │   ├── user_schema.py
│   │   ├── course_schema.py
│   │   └── grade_schema.py
│   │
│   ├── modules/                    # Blueprints modulares
│   │   ├── auth/
│   │   │   ├── __init__.py
│   │   │   ├── routes.py           # Login, Register, Refresh Token
│   │   │   └── services.py         # Lógica de negocio auth
│   │   │
│   │   ├── users/
│   │   │   ├── __init__.py
│   │   │   ├── routes.py           # CRUD Usuarios
│   │   │   └── services.py
│   │   │
│   │   ├── courses/
│   │   │   ├── __init__.py
│   │   │   ├── routes.py
│   │   │   └── services.py
│   │   │
│   │   └── grades/
│   │       ├── __init__.py
│   │       ├── routes.py
│   │       └── services.py
│   │
│   ├── middlewares/                # Decoradores personalizados
│   │   ├── __init__.py
│   │   └── role_required.py        # @admin_required, @teacher_required, etc.
│   │
│   └── utils/                      # Utilidades generales
│       ├── __init__.py
│       ├── error_handlers.py       # Global Error Handler
│       └── responses.py            # Formateadores de respuesta JSON
│
├── migrations/                     # Flask-Migrate
├── tests/                          # Tests unitarios e integración
├── docker-compose.yml
├── requirements.txt
├── .env.example
├── .gitignore
└── run.py                          # Entry point
```


### 1.2 Escalabilidad Futura

**Estrategia de Escalamiento Horizontal:**

**1. Nuevos Módulos: Cada funcionalidad se agrega como Blueprint independiente:**

- ```app/modules/ai/``` - Servicio de IA (recomendaciones, análisis)
- ```app/modules/video/``` - Streaming de video (integración con Vimeo/AWS S3)
- ```app/modules/payments/``` - Pasarela de pagos (Stripe/PayU)


**2. Microservicios (Fase 2):**

- Separar módulos pesados (IA, Video) en servicios independientes
- Comunicación vía API Gateway + Message Queue (RabbitMQ/Redis)


**3. Caching:**

- Redis para sessions, permisos de usuario, datos de cursos frecuentes


**4. Base de Datos:**

- Sharding por institución educativa
- Read Replicas para consultas de reportes.

---
## 2. Modelo de Datos (ERD Lógico)
### 2.1 Diagrama Entidad-Relación

```
┌─────────────────┐
│      User       │
├─────────────────┤
│ id (PK)         │
│ email (UNIQUE)  │
│ password_hash   │
│ role (ENUM)     │◄──┐
│ first_name      │   │
│ last_name       │   │
│ is_active       │   │
│ created_at      │   │
└─────────────────┘   │
         │            │
         │ 1          │
         │            │
         │ *          │
┌─────────────────────┴──┐
│  guardian_student      │ (Tabla intermedia)
├────────────────────────┤
│ guardian_id (FK)       │
│ student_id (FK)        │
│ relationship_type      │
└────────────────────────┘
         │
         │ *
         ▼ 1
┌─────────────────┐
│     Course      │
├─────────────────┤
│ id (PK)         │
│ name            │
│ description     │
│ teacher_id (FK) │──┐
│ grade_level     │  │ 1
│ created_at      │  │
└─────────────────┘  │
         │           │
         │ *         │
         │           │
         │ *         ▼ 1
┌──────────────────────┐   ┌─────────────────┐
│ course_enrollment    │   │      User       │
├──────────────────────┤   │   (Teacher)     │
│ student_id (FK)      │   └─────────────────┘
│ course_id (FK)       │
│ enrolled_at          │
└──────────────────────┘
         │
         │ 1
         │
         │ *
┌─────────────────┐
│      Grade      │
├─────────────────┤
│ id (PK)         │
│ student_id (FK) │
│ course_id (FK)  │
│ teacher_id (FK) │
│ value (DECIMAL) │
│ grade_type      │ (Parcial, Final, Quiz)
│ comments        │
│ created_at      │
└─────────────────┘
```

### 2.2 Modelos SQLAlchemy
``` app/models/user.py ```
```
from app.extensions import db
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime
import enum

class UserRole(enum.Enum):
    STUDENT = "student"
    TEACHER = "teacher"
    ADMIN = "admin"
    GUARDIAN = "guardian"

class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(120), unique=True, nullable=False, index=True)
    password_hash = db.Column(db.String(255), nullable=False)
    role = db.Column(db.Enum(UserRole), nullable=False)
    first_name = db.Column(db.String(50), nullable=False)
    last_name = db.Column(db.String(50), nullable=False)
    is_active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    # Relaciones
    taught_courses = db.relationship('Course', back_populates='teacher', lazy='dynamic')
    enrollments = db.relationship('CourseEnrollment', back_populates='student', lazy='dynamic')
    grades_received = db.relationship('Grade', foreign_keys='Grade.student_id', back_populates='student', lazy='dynamic')
    grades_given = db.relationship('Grade', foreign_keys='Grade.teacher_id', back_populates='teacher', lazy='dynamic')
    
    # Relación recursiva para Acudientes
    wards = db.relationship(
        'GuardianStudent',
        foreign_keys='GuardianStudent.guardian_id',
        back_populates='guardian',
        lazy='dynamic'
    )
    
    def set_password(self, password):
        self.password_hash = generate_password_hash(password)
    
    def check_password(self, password):
        return check_password_hash(self.password_hash, password)
    
    def __repr__(self):
        return f'<User {self.email} ({self.role.value})>'

```