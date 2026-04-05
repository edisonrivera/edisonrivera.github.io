---
title: 'Implementacion de Autorizacion basada en roles con FastAPI'
author: edison
date: 2024-06-26 18:51:00 +0800
categories: ['Python', 'FastAPI']
tags: ['Autenticación', 'JWT']
image:
  path: /assets/images/python/auth/logo.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: ShellShock Logo
---

Veremos una forma de implementar una autenticación basada en roles en FastAPI ⚡


> 🐱‍💻 Repositorio de Github: <https://github.com/edisonrivera/RoleBasedAuth-FastAPI>
{: .prompt-tip }

- Ejemplo de las tablas usadas
- **usuario**

| id           | nickname     | password |
| :----------- | :------------| ------:  |
| bigserial    | varchar(25)  | text     |

- **rol**

| id        | name         |
| :---------| :------------|
| serial4   | varchar(15)  |

- **rol_usuario**

| id           | rol_id     | usuario_id     |
| :----------- | :----------| ------------:  |
| bigserial    | fk(rol.id) | fk(usuario.id) |

1. Definimos nuestras variables de entorno

```python
from pydantic_settings import BaseSettings
from functools import lru_cache
import os


@lru_cache
def get_env_filename() -> str:
    runtime_env = os.getenv("ENV")
    return f".env.{runtime_env}" if runtime_env else ".env"


class EnvironmentSettings(BaseSettings):
    DB_URL: str
    ENVIRONMENT: str
    ALGORITHM: str
    JWT_SECRET: str
    
    class Config:
        env_file: str = get_env_filename()
        env_file_encoding: str = "utf-8"
        
        
@lru_cache
def get_env_vars() -> EnvironmentSettings:
    return EnvironmentSettings()
```


2. Definimos un `Enum` con los **nombres de roles** que tengamos en nuestra base de datos

```python
from enum import StrEnum, unique, auto

@unique
class RoleEnum(StrEnum):
    @staticmethod
    def _generate_next_value_(name, *args):
        return name.upper()
    
    ADMIN = auto()
    USER = auto()
    SUPPORT = auto()
```

> Sobreescribimos el método **`_generate_next_value_`** para que el método **`auto()`** nos devuelva el valor como un string en uppercase (mayúsculas).
{: .prompt-tip }


3. Creamos 2 métodos para **firmar** y **decodear** los JWT

```python
from app.core.config.env_variables import get_env_vars
from app.schemas.auth_schema import JWTPayload, JWTSchema
from fastapi.security import OAuth2PasswordBearer
from fastapi import Depends, HTTPException, status
from jose import jwt, JWTError, ExpiredSignatureError
from typing import Dict, Any

env = get_env_vars()
oauth2_schema = OAuth2PasswordBearer("/api/v1/auth/login")


def sign_jwt(payload: JWTPayload) -> JWTSchema:
    return JWTSchema(access_token=jwt.encode(payload.model_dump(), env.JWT_SECRET, algorithm=env.ALGORITHM))

def decode_jwt(token: str = Depends(oauth2_schema)) -> Dict[str, Any]:
    try:
        if not token:
            raise HTTPException(status_code=status.HTTP_203_NON_AUTHORITATIVE_INFORMATION, 
                                detail="Token not exists")
            
        return jwt.decode(token, env.JWT_SECRET, algorithms=[env.ALGORITHM])
        
    except ExpiredSignatureError:
          raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, 
                                detail="Token Expired")      
            
    except JWTError as e:
        print(e)
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, 
                                detail="Invalid Token")
```

4. Ahora, con una query consultamos los roles que tenga este usuario

```python
from app.persistence.database_config import get_db
from app.persistence.entity.db_entities import UserEntity, RoleEntity
from typing import List
from sqlalchemy import select


class UsuarioRepository:
    def get_roles(self, id: int) -> List[str]:
        with get_db() as db:
            stmt = select(RoleEntity.name).join(RoleEntity.usuarios).filter(UserEntity.id == id)
            return db.execute(stmt).scalars().all()
```


5. En este punto definiremos la clase que controlará el acceso a los enpoints

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from typing import List, Dict, Any
from app.enums.role_enum import RoleEnum
from app.core.jwt import security_jwt
from app.persistence.repository.usuario_repository import UsuarioRepository
from typing_extensions import Annotated, Doc

oauth2_schema = OAuth2PasswordBearer("/api/v1/auth/login")  # -> Login de nuestra API

class PreAuthorize():    
    def __init__(
        self, 
        allowed_roles: Annotated[
            List[RoleEnum], 
            Doc(
                """
                List of roles, indicate which roles are authorized.
                """
            )
        ] = None,
        allow_all: Annotated[
            bool, 
            Doc("""
                Boolean indicating all roles are authorized.
                """)
        ] = False, 
        strict_roles: Annotated[
            bool, 
            Doc("""
                Indicates that a user must have strictly the indicated roles, 
                no more roles, or contain the indicated roles.
                """)
        ] = False
        ):
        self.allowed_roles = allowed_roles
        self.allow_all = allow_all
        self.strict_roles = strict_roles
        self.__usuario_repository = UsuarioRepository()
        
    async def __call__(self, token: str = Depends(oauth2_schema)):
        payload: Dict[str, Any] = security_jwt.decode_jwt(token)
        
        if self.allowed_roles is None and not self.allow_all:
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Nobody can access")
        
        if self.allow_all:
            return True
        
        user_roles: List[str] = self.__usuario_repository.get_roles(payload.get("id"))
        
        allow: bool = all(rol.value in user_roles for rol in self.allowed_roles) if self.strict_roles else \
            any(rol.value in user_roles for rol in self.allowed_roles)

        if not allow:
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="You can't access")
```

> Esta clase recibe 3 parámetros **`allowed_roles`** una lista de `Enums` (los roles que definimos previamente), **`allow_all`** un booleano que indica 
> si cualquier usuario puede acceder y **`strict_roles`** que es un booleano que nos servirá para indicar que el usuario que desee acceder a este endpoint deba tener
> exactamente los roles que se indican.
{: .prompt-tip }


> Usamos `__call__` para que la clase pueda ser tratada como una función cuando sea usada. Esto no es muy útil (Lo veremos después 👀)
{: .prompt-info }

6. Creamos endpoints de prueba para cada rol

```python
from app.model.response_model import ResponseModel
from app.core.security.preauthorize import PreAuthorize
from app.enums.role_enum import RoleEnum
from fastapi import APIRouter, Depends


test = APIRouter(prefix="/test", tags=["Test"])


@test.get("/user", dependencies=[Depends(PreAuthorize(allowed_roles=[RoleEnum.USER, RoleEnum.SUPPORT], strict_roles=True))], response_model=ResponseModel[str],
          response_model_exclude_none=True)
async def user():
    return ResponseModel[str](message="User Dashboard")


@test.get("/admin", dependencies=[Depends(PreAuthorize(allowed_roles=[RoleEnum.ADMIN]))], response_model=ResponseModel[str],
          response_model_exclude_none=True)
async def admin():
    return ResponseModel[str](message="Admin Dashboard")


@test.get("/support", dependencies=[Depends(PreAuthorize(allowed_roles=[RoleEnum.SUPPORT]))], response_model=ResponseModel[str],
          response_model_exclude_none=True)
async def support():
    return ResponseModel[str](message="Support Dashboard")
```

> La clave de esto está en **`dependencies=Depends(PreAuthorize(allowed_roles=[RoleEnum.USER, RoleEnum.SUPPORT], strict_roles=True))]`**. Cuando definimos una clase o función aquí, esta se ejecutará antes de realizar cualquier proceso que esté dentro del endpoint 💪
{: .prompt-tip }


🌟 Ventajas:
1. Fácil de moldear a las necesidades de cada proyecto. Podrían colocar más comprobaciones como si un usuario está **verificado** o no.
2. Excepciones cubiertas fácilmente.
3. Una forma simple e intuitiva de usar e implementar.

⭕ Desventajas:
1. Cada que un usuario visite un endpoint se realizará un query a la base de datos.
2. Si se quiere añadir otro rol en la base de datos también se deberá añadir en el `Enum`.
