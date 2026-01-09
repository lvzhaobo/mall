# Python编码规范

> 📝 **基于PEP 8** + ⭐ **企业专属扩展**

## 1. 命名规范

### 1.1 模块和包命名

#### 📝 通用规范
- 使用**小写字母**，单词间用下划线分隔
- 包名尽量简短，避免使用下划线

```python
# ✅ 正确
import user_service
from utils import string_helper

# ❌ 错误
import UserService
import user-service
```

#### ⭐ 企业专属规范
- 所有企业内部包必须以公司代码开头
- 格式：`{company_code}_{module_name}`

```python
# ✅ 企业规范
from acme_core import config
from acme_utils import logger
from acme_services import user_service

# 说明：acme 为公司代码
```

---

### 1.2 类命名

#### 📝 通用规范
- 使用**大驼峰命名法**（CapWords）
- 异常类以 `Error` 或 `Exception` 结尾

```python
# ✅ 正确
class UserService:
    pass

class DatabaseConnectionError(Exception):
    pass

# ❌ 错误
class user_service:
    pass

class User_Service:
    pass
```

#### ⭐ 企业专属规范
- 业务实体类必须继承 `BaseModel`
- Service类必须以 `Service` 结尾
- DTO类必须以 `DTO` 结尾

```python
# ✅ 企业规范
from acme_core.base import BaseModel

class User(BaseModel):
    """用户实体类，必须继承BaseModel"""
    pass

class UserService:
    """用户服务类，必须以Service结尾"""
    pass

class UserDTO:
    """用户数据传输对象，必须以DTO结尾"""
    pass
```

---

### 1.3 函数和方法命名

#### 📝 通用规范
- 使用**小写字母**，单词间用下划线分隔
- 私有方法使用单下划线前缀

```python
# ✅ 正确
def get_user_by_id(user_id: int):
    pass

def _internal_helper():
    pass

# ❌ 错误
def GetUserById(userId):
    pass

def getUserById(user_id):
    pass
```

#### ⭐ 企业专属规范
- 查询方法必须以 `get_` 或 `find_` 开头
- 列表查询必须以 `list_` 开头
- 保存方法必须使用 `save_` 或 `create_`
- 更新方法必须使用 `update_`
- 删除方法必须使用 `delete_` 或 `remove_`

```python
# ✅ 企业规范
class UserService:
    def get_user_by_id(self, user_id: int) -> Optional[User]:
        """查询单个用户"""
        pass
    
    def list_users(self, keyword: str = None) -> List[User]:
        """查询用户列表"""
        pass
    
    def create_user(self, user_dto: UserDTO) -> User:
        """创建用户"""
        pass
    
    def update_user(self, user_id: int, user_dto: UserDTO) -> None:
        """更新用户"""
        pass
    
    def delete_user(self, user_id: int) -> None:
        """删除用户"""
        pass

# ❌ 错误 - 不符合企业规范
def query_user(user_id: int):  # 应该用 get_user_by_id
    pass

def fetch_users():  # 应该用 list_users
    pass
```

---

### 1.4 变量命名

#### 📝 通用规范
- 使用**小写字母**，单词间用下划线分隔
- 常量使用全大写字母
- 避免单字母变量名（循环除外）

```python
# ✅ 正确
user_name = "张三"
MAX_RETRY_COUNT = 3
user_list = []

for i in range(10):
    print(i)

# ❌ 错误
userName = "张三"
n = "张三"
```

#### ⭐ 企业专属规范
- 布尔变量必须以 `is_`、`has_`、`can_` 开头
- 临时变量禁止使用 `temp`、`tmp`，必须使用有意义的名称
- 集合变量必须使用复数形式或带 `_list`、`_dict` 后缀

```python
# ✅ 企业规范
is_active = True
has_permission = False
can_delete = True

user_ids = [1, 2, 3]
user_list = []
user_dict = {}

# ❌ 错误 - 不符合企业规范
active = True  # 应该用 is_active
permission = False  # 应该用 has_permission
temp = []  # 禁止使用temp
users = []  # 建议用 user_list 更明确
```

---

## 2. 代码格式规范

### 2.1 缩进和空格

#### 📝 通用规范（PEP 8）
- 使用 **4个空格** 缩进
- 不使用Tab键
- 运算符两侧添加空格

```python
# ✅ 正确
def calculate_total(price, quantity):
    total = price * quantity
    return total

# ❌ 错误
def calculate_total(price,quantity):
    total=price*quantity
    return total
```

---

### 2.2 行长度限制

#### 📝 通用规范
- 每行最多 **79个字符**
- 文档字符串和注释最多 **72个字符**

#### ⭐ 企业专属规范
- 允许放宽到 **120个字符**（针对现代宽屏）
- 但必须使用工具自动检查

```python
# ✅ 企业规范 - 120字符限制
def create_user_with_details(username: str, email: str, phone: str, 
                             address: str, department: str) -> User:
    """创建用户（允许120字符）"""
    pass
```

---

### 2.3 导入规范

#### 📝 通用规范
- 每个导入独占一行
- 分组排序：标准库、第三方库、本地模块
- 组间空一行

```python
# ✅ 正确
import os
import sys
from typing import List, Dict

import requests
from flask import Flask

from acme_core import config
from acme_utils import logger
```

#### ⭐ 企业专属规范
- 必须使用 `isort` 自动排序
- 禁止使用 `from module import *`
- 企业内部包导入必须使用绝对导入

```python
# ✅ 企业规范
from acme_core.models import User  # 绝对导入
from acme_services.user_service import UserService

# ❌ 错误
from acme_core.models import *  # 禁止使用 *
from ..models import User  # 禁止相对导入企业包
```

---

## 3. 类型注解规范

### 3.1 基本类型注解

#### 📝 通用规范
```python
from typing import List, Dict, Optional, Union

def get_user(user_id: int) -> Optional[User]:
    """可能返回None"""
    pass

def list_users(keyword: str = None) -> List[User]:
    """返回用户列表"""
    pass

def get_user_info(user_id: int) -> Dict[str, str]:
    """返回字典"""
    pass
```

#### ⭐ 企业专属规范
- 所有公共方法**必须**添加类型注解
- 所有Service方法**必须**添加返回类型
- 复杂类型必须使用 TypedDict 或 dataclass

```python
# ✅ 企业规范 - 必须添加类型注解
from typing import List, Optional, TypedDict

class UserDict(TypedDict):
    """企业要求：复杂字典必须定义类型"""
    user_id: int
    username: str
    email: str

class UserService:
    def get_user_by_id(self, user_id: int) -> Optional[User]:
        """必须添加返回类型注解"""
        pass
    
    def list_users(self) -> List[User]:
        """必须添加返回类型注解"""
        pass
    
    def get_user_dict(self, user_id: int) -> UserDict:
        """复杂字典使用TypedDict"""
        pass

# ❌ 错误 - 不符合企业规范
class UserService:
    def get_user_by_id(self, user_id):  # 缺少类型注解
        pass
```

---

## 4. 文档字符串规范

### 4.1 函数文档字符串

#### 📝 通用规范（Google风格）
```python
def calculate_discount(price: float, discount_rate: float) -> float:
    """计算折扣后价格
    
    Args:
        price: 原价
        discount_rate: 折扣率（0-1之间）
    
    Returns:
        折扣后的价格
    
    Raises:
        ValueError: 当discount_rate不在0-1之间时
    """
    if not 0 <= discount_rate <= 1:
        raise ValueError("折扣率必须在0-1之间")
    return price * (1 - discount_rate)
```

#### ⭐ 企业专属规范
- 所有公共类和方法**必须**添加文档字符串
- 必须包含：功能说明、参数、返回值、异常
- 使用中文编写（特殊术语可用英文）

```python
# ✅ 企业规范
class UserService:
    """用户服务类
    
    提供用户的增删改查功能，包括：
    - 用户创建和更新
    - 用户查询（单个和列表）
    - 用户删除（软删除）
    
    Note:
        所有删除操作均为软删除，不会物理删除数据
    """
    
    def create_user(self, user_dto: UserDTO) -> User:
        """创建用户
        
        Args:
            user_dto: 用户数据传输对象，包含用户基本信息
        
        Returns:
            创建成功的用户对象
        
        Raises:
            ValidationError: 当用户数据校验失败时
            DuplicateError: 当用户名已存在时
        
        Example:
            >>> user_dto = UserDTO(username="zhangsan", email="zhangsan@company.com")
            >>> user = user_service.create_user(user_dto)
        """
        pass

# ❌ 错误 - 不符合企业规范
def create_user(user_dto):  # 没有文档字符串
    pass
```

---

## 5. 代码组织规范

### 5.1 模块结构

#### ⭐ 企业专属规范
- 标准模块结构顺序：

```python
"""模块文档字符串"""

# 1. 导入部分
import os
import sys

from typing import List

# 2. 常量定义
MAX_RETRY_COUNT = 3
DEFAULT_TIMEOUT = 30

# 3. 异常定义
class CustomError(Exception):
    pass

# 4. 类定义
class UserService:
    pass

# 5. 函数定义
def helper_function():
    pass

# 6. 主程序入口（如果有）
if __name__ == "__main__":
    main()
```

---

### 5.2 类结构

#### ⭐ 企业专属规范
- 标准类结构顺序：

```python
class UserService:
    """类文档字符串"""
    
    # 1. 类变量
    default_page_size = 10
    
    # 2. __init__ 方法
    def __init__(self, db_session):
        self.db_session = db_session
        self._cache = {}
    
    # 3. 公共方法（按功能分组）
    def create_user(self, user_dto: UserDTO) -> User:
        pass
    
    def get_user_by_id(self, user_id: int) -> Optional[User]:
        pass
    
    def list_users(self) -> List[User]:
        pass
    
    # 4. 私有方法
    def _validate_user(self, user: User) -> bool:
        pass
    
    def _build_query(self):
        pass
    
    # 5. 特殊方法
    def __str__(self):
        return f"UserService(session={self.db_session})"
```

---

## 6. 企业专属最佳实践

### 6.1 配置管理

#### ⭐ 企业专属规范
```python
# ✅ 企业规范 - 使用配置类
from acme_core.config import BaseConfig

class Config(BaseConfig):
    """应用配置类，必须继承BaseConfig"""
    
    # 数据库配置
    DATABASE_URL = "postgresql://localhost/mydb"
    
    # Redis配置
    REDIS_HOST = "localhost"
    REDIS_PORT = 6379
    
    # 业务配置
    MAX_LOGIN_ATTEMPTS = 5
    SESSION_TIMEOUT = 3600

# ❌ 错误 - 禁止硬编码
def connect_db():
    conn = psycopg2.connect("postgresql://localhost/mydb")  # 禁止硬编码
```

---

### 6.2 日志规范

#### ⭐ 企业专属规范
```python
# ✅ 企业规范 - 使用企业日志工具
from acme_utils.logger import get_logger

logger = get_logger(__name__)

class UserService:
    def create_user(self, user_dto: UserDTO) -> User:
        logger.info(f"开始创建用户，username: {user_dto.username}")
        
        try:
            user = self._save_user(user_dto)
            logger.info(f"用户创建成功，user_id: {user.id}")
            return user
        except Exception as e:
            logger.error(f"用户创建失败，username: {user_dto.username}", 
                        exc_info=True)
            raise

# ❌ 错误 - 禁止使用print
def create_user(user_dto):
    print(f"创建用户: {user_dto.username}")  # 禁止使用print
```

---

### 6.3 异常处理

#### ⭐ 企业专属规范
```python
# ✅ 企业规范 - 使用企业自定义异常
from acme_core.exceptions import (
    BusinessException,
    ValidationError,
    DataNotFoundError
)

class UserService:
    def get_user_by_id(self, user_id: int) -> User:
        if user_id <= 0:
            raise ValidationError("用户ID必须大于0")
        
        user = self.db_session.query(User).filter_by(id=user_id).first()
        if not user:
            raise DataNotFoundError(f"用户不存在，user_id: {user_id}")
        
        return user

# ❌ 错误 - 不使用内置异常
def get_user_by_id(user_id: int) -> User:
    if not user:
        raise Exception("用户不存在")  # 禁止使用通用Exception
```

---

## 7. 代码检查工具配置

### 7.1 black配置

#### ⭐ 企业专属配置
```toml
# pyproject.toml
[tool.black]
line-length = 120  # 企业规范：120字符
target-version = ['py38', 'py39', 'py310']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.venv
  | build
  | dist
)/
'''
```

---

### 7.2 pylint配置

#### ⭐ 企业专属配置
```ini
# .pylintrc
[MASTER]
init-hook='import sys; sys.path.append("src")'

[FORMAT]
max-line-length=120  # 企业规范：120字符
indent-string='    '  # 4个空格

[BASIC]
# 企业命名规范
class-naming-style=PascalCase
function-naming-style=snake_case
const-naming-style=UPPER_CASE
variable-naming-style=snake_case

[MESSAGES CONTROL]
disable=C0111  # 允许某些情况不写文档字符串
```

---

### 7.3 isort配置

#### ⭐ 企业专属配置
```toml
# pyproject.toml
[tool.isort]
profile = "black"
line_length = 120
known_first_party = ["acme_core", "acme_utils", "acme_services"]
sections = ["FUTURE", "STDLIB", "THIRDPARTY", "FIRSTPARTY", "LOCALFOLDER"]
```

---

## 8. 检查清单

### 代码提交前检查
- [ ] 命名符合PEP 8规范
- [ ] 企业命名规范已遵守
- [ ] 类型注解完整
- [ ] 文档字符串完整
- [ ] black格式化完成
- [ ] isort导入排序完成
- [ ] pylint检查通过（分数>8.0）
- [ ] 企业自定义异常使用正确
- [ ] 配置无硬编码
- [ ] 日志使用企业工具

---

## 9. 企业规范速查表

| 类型 | 企业规范 | 示例 |
|------|---------|------|
| 包命名 | {company_code}_{module} | acme_core |
| 实体类 | 必须继承BaseModel | class User(BaseModel) |
| Service类 | 必须以Service结尾 | class UserService |
| 查询方法 | get_/find_/list_ | get_user_by_id |
| 布尔变量 | is_/has_/can_ | is_active |
| 行长度 | 120字符 | - |
| 类型注解 | 公共方法必须有 | def get(id: int) -> User |
| 文档字符串 | 公共类方法必须有 | """创建用户...""" |
| 日志工具 | 使用企业logger | from acme_utils.logger |
| 异常类 | 使用企业自定义 | from acme_core.exceptions |
| 配置管理 | 继承BaseConfig | class Config(BaseConfig) |
