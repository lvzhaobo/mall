# Python开发规范体系

## 📚 规范体系概览

本目录包含完整的企业级Python开发规范体系，适用于大型企业的多层级、多团队协作场景。

## 📁 规范分类

### 1. 基础规范（必读）
- [Python编码规范](./01-coding-standards.md) - PEP 8扩展、命名、格式、类型注解
- [代码质量规范](./02-code-quality.md) - 代码复杂度、可读性、重构原则
- [异常处理规范](./03-exception-handling.md) - 异常分类、处理原则、自定义异常

### 2. 框架规范
- [Django开发规范](./04-django-standards.md) - 项目结构、Model、View、ORM规范
- [Flask开发规范](./05-flask-standards.md) - Blueprint、路由、请求处理
- [FastAPI开发规范](./06-fastapi-standards.md) - 异步编程、依赖注入、性能优化

### 3. 数据处理规范
- [数据库操作规范](./07-database-standards.md) - SQLAlchemy、连接池、事务管理
- [数据分析规范](./08-data-analysis.md) - Pandas、NumPy最佳实践
- [异步编程规范](./09-async-standards.md) - asyncio、协程、并发控制

### 4. 测试规范
- [单元测试规范](./10-unit-test.md) - pytest、Mock、覆盖率
- [集成测试规范](./11-integration-test.md) - 测试策略、环境准备
- [性能测试规范](./12-performance-test.md) - 压测工具、性能指标

### 5. 部署运维规范
- [日志规范](./13-logging-standards.md) - logging模块、日志级别、格式化
- [配置管理规范](./14-config-management.md) - 环境变量、配置文件、密钥管理
- [Docker规范](./15-docker-standards.md) - Dockerfile、镜像优化、容器编排

### 6. 团队协作规范
- [Git使用规范](./16-git-standards.md) - 分支管理、提交规范、Code Review
- [文档编写规范](./17-documentation.md) - docstring、Sphinx、API文档
- [依赖管理规范](./18-dependency-management.md) - requirements.txt、Poetry、虚拟环境

## 🎯 适用场景

### 小型项目（3人以下）
**必读规范：**
- Python编码规范
- 异常处理规范
- 日志规范
- Git使用规范

### 中型项目（3-10人）
**必读规范：**
- 基础规范（全部）
- 对应框架规范（Django/Flask/FastAPI）
- 数据库操作规范
- 单元测试规范
- 团队协作规范

### 大型项目（10人以上）
**必读规范：**
- 全部规范

## 📖 学习路径

### 新员工（第1周）
1. Python编码规范
2. 异常处理规范
3. 对应框架规范
4. Git使用规范

### 初级开发（第2-4周）
1. 代码质量规范
2. 数据库操作规范
3. 单元测试规范
4. 日志规范
5. 配置管理规范

### 中级开发（1-3个月）
1. 异步编程规范
2. 数据分析规范
3. 性能测试规范
4. Docker规范
5. 依赖管理规范

### 高级开发（3-6个月）
1. 集成测试规范
2. 文档编写规范
3. 性能优化
4. 架构设计

## 🔍 快速查找

### 按问题场景查找

| 问题 | 参考规范 |
|------|---------|
| 函数、类如何命名？ | Python编码规范 |
| 如何处理异常？ | 异常处理规范 |
| Django Model怎么写？ | Django开发规范 |
| 如何使用ORM？ | 数据库操作规范 |
| 如何写单元测试？ | 单元测试规范 |
| 日志如何打印？ | 日志规范 |
| 如何管理配置？ | 配置管理规范 |
| 如何使用异步？ | 异步编程规范 |
| Git分支如何管理？ | Git使用规范 |

## ⚡ 常用速查

### 命名速查
```python
# 类名：大驼峰
class UserService:
    pass

# 函数名：小写下划线
def get_user_by_id():
    pass

# 常量：全大写下划线
MAX_RETRY_COUNT = 3

# 变量：小写下划线
user_name = "张三"

# 私有属性：单下划线前缀
_internal_cache = {}
```

### 类型注解速查
```python
from typing import List, Dict, Optional

def get_users(user_ids: List[int]) -> List[Dict[str, str]]:
    """查询用户列表"""
    pass

def find_user(user_id: int) -> Optional[User]:
    """查询单个用户，可能返回None"""
    pass
```

### 异常处理速查
```python
# ✅ 正确
try:
    result = process_data(data)
except ValueError as e:
    logger.error(f"数据处理失败: {e}", exc_info=True)
    raise BusinessException("数据格式错误") from e

# ❌ 错误
try:
    result = process_data(data)
except Exception:
    pass  # 空捕获
```

## 📊 规范检查清单

### 代码提交前检查
- [ ] 命名符合PEP 8规范
- [ ] 添加类型注解
- [ ] 代码格式化完成（black）
- [ ] 通过pylint检查
- [ ] 导入语句排序（isort）
- [ ] 异常处理正确
- [ ] 日志记录完整
- [ ] 单元测试通过
- [ ] 代码审查通过

### Code Review检查
- [ ] 业务逻辑正确
- [ ] 代码可读性良好
- [ ] 性能符合要求
- [ ] 安全问题已处理
- [ ] 异常处理完善
- [ ] 资源正确释放
- [ ] 日志级别合理
- [ ] 测试覆盖充分

## 🛠 开发工具

### 代码格式化
```bash
# black - 代码格式化
pip install black
black your_file.py

# isort - 导入排序
pip install isort
isort your_file.py
```

### 代码检查
```bash
# pylint - 代码质量检查
pip install pylint
pylint your_file.py

# flake8 - 代码风格检查
pip install flake8
flake8 your_file.py

# mypy - 类型检查
pip install mypy
mypy your_file.py
```

### 测试工具
```bash
# pytest - 单元测试
pip install pytest pytest-cov
pytest --cov=your_module tests/

# tox - 多环境测试
pip install tox
tox
```

## 📞 技术支持

- 📧 技术规范团队：python-dev@company.com
- 💬 企业微信群：Python开发规范交流群
- 📖 Wiki：http://wiki.company.com/python-standards

## 🔄 更新记录

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2024-01-07 | 初始版本，包含18个规范文档 |

---

**注意：** 
- ⭐ 标记表示**企业专属规范**，需严格遵守
- 📝 标记表示基于行业标准的通用规范
- 本规范体系会持续更新，请定期查看最新版本
