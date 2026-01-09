# 异常处理规范

> 📝 **Python标准异常** + ⭐ **企业自定义异常体系**

## 1. 异常分类

### 1.1 Python内置异常

```python
# 常用内置异常
ValueError          # 值错误
TypeError          # 类型错误
KeyError           # 字典键不存在
IndexError         # 索引超出范围
AttributeError     # 属性不存在
FileNotFoundError  # 文件不存在
ConnectionError    # 连接错误
TimeoutError       # 超时错误
```

---

### 1.2 企业自定义异常体系

#### ⭐ 企业专属规范
企业必须使用统一的异常基类和分类体系：

```python
"""
企业异常体系
acme_core/exceptions.py
"""
from typing import Optional, Dict, Any


class BaseException(Exception):
    """企业异常基类
    
    所有企业自定义异常必须继承此类
    """
    
    def __init__(
        self, 
        message: str, 
        code: str = "UNKNOWN_ERROR",
        details: Optional[Dict[str, Any]] = None
    ):
        self.message = message
        self.code = code
        self.details = details or {}
        super().__init__(self.message)
    
    def to_dict(self) -> Dict[str, Any]:
        """转换为字典格式"""
        return {
            "code": self.code,
            "message": self.message,
            "details": self.details
        }


class BusinessException(BaseException):
    """业务异常
    
    用于业务规则不满足的场景
    """
    def __init__(self, message: str, code: str = "BUSINESS_ERROR", **kwargs):
        super().__init__(message, code, **kwargs)


class ValidationError(BaseException):
    """参数校验异常
    
    用于参数校验失败的场景
    """
    def __init__(self, message: str, field: str = None, **kwargs):
        details = kwargs.get('details', {})
        if field:
            details['field'] = field
        super().__init__(message, code="VALIDATION_ERROR", details=details)


class DataNotFoundError(BaseException):
    """数据不存在异常
    
    用于查询数据不存在的场景
    """
    def __init__(self, message: str, resource: str = None, **kwargs):
        details = kwargs.get('details', {})
        if resource:
            details['resource'] = resource
        super().__init__(message, code="DATA_NOT_FOUND", details=details)


class PermissionDeniedError(BaseException):
    """权限不足异常
    
    用于权限校验失败的场景
    """
    def __init__(self, message: str = "权限不足", **kwargs):
        super().__init__(message, code="PERMISSION_DENIED", **kwargs)


class SystemError(BaseException):
    """系统异常
    
    用于系统级错误，如数据库连接失败、第三方服务异常等
    """
    def __init__(self, message: str = "系统异常，请稍后重试", **kwargs):
        super().__init__(message, code="SYSTEM_ERROR", **kwargs)


class DuplicateError(BaseException):
    """重复数据异常
    
    用于唯一性约束冲突的场景
    """
    def __init__(self, message: str, field: str = None, **kwargs):
        details = kwargs.get('details', {})
        if field:
            details['field'] = field
        super().__init__(message, code="DUPLICATE_ERROR", details=details)
```

---

## 2. 异常处理原则

### 2.1 不要吞掉异常

#### ❌ 错误示例
```python
# 示例1：空except块
try:
    user = user_service.delete_user(user_id)
except Exception:
    pass  # 吞掉异常

# 示例2：只记录不抛出
try:
    order = order_service.create_order(order_dto)
except Exception as e:
    print(f"错误: {e}")  # 只打印，不抛出
```

#### ✅ 正确示例
```python
# 记录日志并重新抛出
try:
    user = user_service.delete_user(user_id)
except Exception as e:
    logger.error(f"删除用户失败，user_id: {user_id}", exc_info=True)
    raise SystemError("删除用户失败") from e
```

---

### 2.2 捕获具体异常

#### ❌ 错误示例
```python
try:
    with open(file_path) as f:
        data = f.read()
except Exception as e:  # 过于宽泛
    logger.error("文件读取失败", exc_info=True)
```

#### ✅ 正确示例
```python
try:
    with open(file_path) as f:
        data = f.read()
except FileNotFoundError as e:
    logger.error(f"文件不存在: {file_path}", exc_info=True)
    raise DataNotFoundError(f"文件不存在: {file_path}") from e
except PermissionError as e:
    logger.error(f"文件访问权限不足: {file_path}", exc_info=True)
    raise PermissionDeniedError("文件访问权限不足") from e
```

---

### 2.3 使用上下文管理器

#### ✅ 推荐方式
```python
# 自动关闭资源
with open(file_path) as f:
    data = f.read()

# 数据库连接
with db.session() as session:
    user = session.query(User).filter_by(id=user_id).first()
```

---

## 3. Service层异常处理

### 3.1 标准异常处理模式

#### ⭐ 企业专属规范
Service层必须遵循以下异常处理模式：

```python
from typing import Optional
from acme_core.exceptions import (
    BusinessException,
    ValidationError,
    DataNotFoundError,
    DuplicateError,
    SystemError
)
from acme_utils.logger import get_logger

logger = get_logger(__name__)


class UserService:
    """用户服务"""
    
    def __init__(self, db_session, redis_client):
        self.db = db_session
        self.redis = redis_client
    
    def create_user(self, user_dto: UserDTO) -> User:
        """创建用户
        
        Raises:
            ValidationError: 参数校验失败
            DuplicateError: 用户名已存在
            SystemError: 系统异常
        """
        # 1. 参数校验
        self._validate_user(user_dto)
        
        # 2. 检查重复
        existing_user = self.db.query(User).filter_by(
            username=user_dto.username
        ).first()
        
        if existing_user:
            logger.warning(f"用户名已存在: {user_dto.username}")
            raise DuplicateError(
                message="用户名已存在",
                field="username",
                details={"username": user_dto.username}
            )
        
        # 3. 保存用户
        try:
            user = User(**user_dto.dict())
            self.db.add(user)
            self.db.commit()
            self.db.refresh(user)
            
            logger.info(f"用户创建成功，user_id: {user.id}")
            return user
            
        except Exception as e:
            self.db.rollback()
            logger.error(
                f"用户创建失败，username: {user_dto.username}",
                exc_info=True
            )
            raise SystemError("用户创建失败，请稍后重试") from e
    
    def get_user_by_id(self, user_id: int) -> User:
        """查询用户
        
        Raises:
            ValidationError: 参数校验失败
            DataNotFoundError: 用户不存在
        """
        # 参数校验
        if user_id <= 0:
            raise ValidationError(
                message="用户ID必须大于0",
                field="user_id"
            )
        
        # 查询用户
        user = self.db.query(User).filter_by(id=user_id).first()
        if not user:
            logger.warning(f"用户不存在，user_id: {user_id}")
            raise DataNotFoundError(
                message=f"用户不存在",
                resource="user",
                details={"user_id": user_id}
            )
        
        return user
    
    def _validate_user(self, user_dto: UserDTO) -> None:
        """校验用户数据
        
        Raises:
            ValidationError: 校验失败
        """
        errors = []
        
        if not user_dto.username:
            errors.append("用户名不能为空")
        elif len(user_dto.username) < 3:
            errors.append("用户名至少3个字符")
        
        if not user_dto.email:
            errors.append("邮箱不能为空")
        elif '@' not in user_dto.email:
            errors.append("邮箱格式不正确")
        
        if errors:
            raise ValidationError(
                message="; ".join(errors),
                details={"errors": errors}
            )
```

---

## 4. API层异常处理

### 4.1 Flask全局异常处理

#### ⭐ 企业专属规范
```python
"""
Flask全局异常处理器
app/error_handlers.py
"""
from flask import Flask, jsonify
from acme_core.exceptions import (
    BaseException,
    ValidationError,
    DataNotFoundError,
    PermissionDeniedError,
    BusinessException,
    SystemError
)
from acme_utils.logger import get_logger

logger = get_logger(__name__)


def register_error_handlers(app: Flask) -> None:
    """注册全局异常处理器"""
    
    @app.errorhandler(ValidationError)
    def handle_validation_error(error: ValidationError):
        """参数校验异常"""
        logger.warning(f"参数校验失败: {error.message}")
        return jsonify(error.to_dict()), 400
    
    @app.errorhandler(DataNotFoundError)
    def handle_not_found_error(error: DataNotFoundError):
        """数据不存在异常"""
        logger.warning(f"数据不存在: {error.message}")
        return jsonify(error.to_dict()), 404
    
    @app.errorhandler(PermissionDeniedError)
    def handle_permission_error(error: PermissionDeniedError):
        """权限不足异常"""
        logger.warning(f"权限不足: {error.message}")
        return jsonify(error.to_dict()), 403
    
    @app.errorhandler(DuplicateError)
    def handle_duplicate_error(error: DuplicateError):
        """重复数据异常"""
        logger.warning(f"数据重复: {error.message}")
        return jsonify(error.to_dict()), 409
    
    @app.errorhandler(BusinessException)
    def handle_business_error(error: BusinessException):
        """业务异常"""
        logger.warning(f"业务异常: {error.message}")
        return jsonify(error.to_dict()), 400
    
    @app.errorhandler(SystemError)
    def handle_system_error(error: SystemError):
        """系统异常"""
        logger.error(f"系统异常: {error.message}", exc_info=True)
        return jsonify({
            "code": "SYSTEM_ERROR",
            "message": "系统异常，请稍后重试"
        }), 500
    
    @app.errorhandler(Exception)
    def handle_unexpected_error(error: Exception):
        """未知异常"""
        logger.error("未知异常", exc_info=True)
        return jsonify({
            "code": "UNKNOWN_ERROR",
            "message": "系统异常，请稍后重试"
        }), 500
```

---

### 4.2 FastAPI全局异常处理

#### ⭐ 企业专属规范
```python
"""
FastAPI全局异常处理器
app/exception_handlers.py
"""
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from acme_core.exceptions import (
    ValidationError,
    DataNotFoundError,
    PermissionDeniedError,
    BusinessException,
    SystemError
)
from acme_utils.logger import get_logger

logger = get_logger(__name__)


def register_exception_handlers(app: FastAPI) -> None:
    """注册异常处理器"""
    
    @app.exception_handler(ValidationError)
    async def validation_error_handler(request: Request, exc: ValidationError):
        logger.warning(f"参数校验失败: {exc.message}")
        return JSONResponse(
            status_code=400,
            content=exc.to_dict()
        )
    
    @app.exception_handler(DataNotFoundError)
    async def not_found_error_handler(request: Request, exc: DataNotFoundError):
        logger.warning(f"数据不存在: {exc.message}")
        return JSONResponse(
            status_code=404,
            content=exc.to_dict()
        )
    
    @app.exception_handler(PermissionDeniedError)
    async def permission_error_handler(request: Request, exc: PermissionDeniedError):
        logger.warning(f"权限不足: {exc.message}")
        return JSONResponse(
            status_code=403,
            content=exc.to_dict()
        )
    
    @app.exception_handler(SystemError)
    async def system_error_handler(request: Request, exc: SystemError):
        logger.error(f"系统异常: {exc.message}", exc_info=True)
        return JSONResponse(
            status_code=500,
            content={
                "code": "SYSTEM_ERROR",
                "message": "系统异常，请稍后重试"
            }
        )
    
    @app.exception_handler(Exception)
    async def general_error_handler(request: Request, exc: Exception):
        logger.error("未知异常", exc_info=True)
        return JSONResponse(
            status_code=500,
            content={
                "code": "UNKNOWN_ERROR",
                "message": "系统异常，请稍后重试"
            }
        )
```

---

## 5. 异常日志规范

### 5.1 日志级别选择

#### ⭐ 企业专属规范

| 异常类型 | 日志级别 | 是否记录堆栈 | 说明 |
|---------|---------|-------------|------|
| ValidationError | WARNING | 否 | 参数校验失败 |
| DataNotFoundError | WARNING | 否 | 数据不存在 |
| PermissionDeniedError | WARNING | 否 | 权限不足 |
| DuplicateError | WARNING | 否 | 数据重复 |
| BusinessException | WARNING | 否 | 业务规则不满足 |
| SystemError | ERROR | 是 | 系统异常 |
| Exception | ERROR | 是 | 未知异常 |

---

### 5.2 日志记录规范

#### ⭐ 企业专属规范
```python
from acme_utils.logger import get_logger

logger = get_logger(__name__)

# ✅ 正确 - 业务异常使用WARNING
try:
    user = user_service.get_user_by_id(user_id)
except DataNotFoundError as e:
    logger.warning(f"用户不存在，user_id: {user_id}")
    raise

# ✅ 正确 - 系统异常使用ERROR并记录堆栈
try:
    result = external_api.call()
except Exception as e:
    logger.error(
        f"调用外部API失败，url: {api_url}",
        exc_info=True  # 记录完整堆栈
    )
    raise SystemError("外部服务异常") from e

# ❌ 错误 - 业务异常不应该用ERROR
try:
    user = user_service.get_user_by_id(user_id)
except DataNotFoundError as e:
    logger.error(f"用户不存在", exc_info=True)  # 不应该用ERROR
```

---

## 6. 异步代码异常处理

### 6.1 异步函数异常处理

#### ⭐ 企业专属规范
```python
import asyncio
from typing import Optional
from acme_core.exceptions import SystemError, DataNotFoundError
from acme_utils.logger import get_logger

logger = get_logger(__name__)


class AsyncUserService:
    """异步用户服务"""
    
    async def get_user_by_id(self, user_id: int) -> Optional[User]:
        """异步查询用户"""
        try:
            # 异步数据库查询
            async with self.db.session() as session:
                result = await session.execute(
                    select(User).filter_by(id=user_id)
                )
                user = result.scalar_one_or_none()
                
                if not user:
                    raise DataNotFoundError(
                        message="用户不存在",
                        resource="user",
                        details={"user_id": user_id}
                    )
                
                return user
                
        except DataNotFoundError:
            raise
        except Exception as e:
            logger.error(
                f"查询用户失败，user_id: {user_id}",
                exc_info=True
            )
            raise SystemError("查询用户失败") from e
    
    async def batch_get_users(self, user_ids: list[int]) -> list[User]:
        """批量查询用户（并发）"""
        tasks = [self.get_user_by_id(uid) for uid in user_ids]
        
        try:
            # 并发执行，收集所有结果和异常
            results = await asyncio.gather(*tasks, return_exceptions=True)
            
            # 处理结果
            users = []
            for i, result in enumerate(results):
                if isinstance(result, Exception):
                    logger.warning(
                        f"查询用户失败，user_id: {user_ids[i]}, "
                        f"error: {result}"
                    )
                else:
                    users.append(result)
            
            return users
            
        except Exception as e:
            logger.error("批量查询用户失败", exc_info=True)
            raise SystemError("批量查询失败") from e
```

---

## 7. 检查清单

### 异常处理检查
- [ ] 使用企业自定义异常
- [ ] 不吞掉异常
- [ ] 捕获具体异常
- [ ] 记录异常日志
- [ ] 日志级别正确
- [ ] 系统异常记录堆栈
- [ ] 异常信息包含关键参数
- [ ] 使用上下文管理器
- [ ] API层有全局异常处理
- [ ] 异步代码正确处理异常

---

## 8. 企业异常规范速查表

| 场景 | 使用异常 | HTTP状态码 | 日志级别 |
|------|---------|-----------|---------|
| 参数校验失败 | ValidationError | 400 | WARNING |
| 数据不存在 | DataNotFoundError | 404 | WARNING |
| 权限不足 | PermissionDeniedError | 403 | WARNING |
| 数据重复 | DuplicateError | 409 | WARNING |
| 业务规则不满足 | BusinessException | 400 | WARNING |
| 数据库异常 | SystemError | 500 | ERROR |
| 第三方服务异常 | SystemError | 500 | ERROR |
| 未知异常 | SystemError | 500 | ERROR |
