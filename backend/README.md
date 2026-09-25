backend/
│
├── app/
│   ├── __init__.py
│   ├── main.py
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── __init__.py
│   │       ├── router.py
│   │       │
│   │       ├── auth.py
│   │       ├── users.py
│   │       ├── workouts.py
│   │       ├── exercises.py
│   │       └── ...
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py
│   │   ├── security.py
│   │   └── dependencies.py
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── workout.py
│   │   └── exercise.py
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── workout.py
│   │   └── exercise.py
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── workout.py
│   │   └── user.py
│   │
│   ├── repositories/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── workout.py
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── session.py
│   │   └── base.py
│   │
│   └── utils/
│       ├── __init__.py
│       └── ...
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── api/
│   │   ├── test_auth.py
│   │   ├── test_users.py
│   │   └── test_workouts.py
│   └── services/
│       └── ...
│
├── migrations/
│   └── ...
│
├── .env
├── .env.example
├── .gitignore
├── .python-version
├── pyproject.toml
├── README.md
└── uv.lock
