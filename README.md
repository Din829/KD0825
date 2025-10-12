# 通用Agent框架使用指南

[![Python Version](https://img.shields.io/badge/python-3.8%2B-blue.svg)](https://python.org)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Code Style](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

本框架是一个完全泛用化的Agent开发平台，提供了核心能力和定制接口。支持快速构建智能对话助手、数据分析工具、自动化脚本等多种应用场景。

## ✨ 特性

- 🚀 **快速部署** - 一键启动，开箱即用
- 🔧 **高度可定制** - 支持自定义工具、提示词和品牌
- 🌐 **多模型支持** - 兼容OpenAI、Claude等主流AI模型
- 📊 **内置工具集** - 文件操作、代码执行、数据搜索等
- 🎨 **界面定制** - 支持自定义LOGO、颜色和提示信息
- 📝 **详细日志** - 完整的操作记录和调试信息

## 🚀 快速开始

### 环境要求

- Python 3.8 或更高版本
- pip 包管理器
- 有效的AI模型API密钥（OpenAI、Claude等）

### 安装步骤

1. **克隆项目**
```bash
git clone https://github.com/Din829/KD0825.git
cd KD0825
```

2. **安装依赖**
```bash
# 推荐使用虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 安装依赖包
pip install -r requirements.txt
```

3. **配置环境**
```bash
# 复制环境配置文件
cp .env.example .env

# 编辑.env文件，添加你的API密钥
# DBRHEO_MODEL=gpt-4  # 推荐使用GPT-4
# OPENAI_API_KEY=your_openai_api_key_here
```

4. **启动Agent**
```bash
python packages/cli/cli.py
```

### 首次使用示例

启动后，你可以尝试以下命令：

```
# 查看帮助信息
/help

# 文件操作示例
请帮我读取当前目录下的README.md文件

# 代码执行示例
请帮我写一个Python脚本计算1到100的和

# 数据分析示例
请分析这个CSV文件的基本统计信息
```

> **💡 提示**: 不推荐使用Claude模型，因为没有做提示词缓存，消耗量较大。推荐使用GPT-4或GPT-3.5-turbo。

## 核心定制功能

### 1. 项目提示词配置

创建 `PROJECT.md` 文件在项目根目录(已创建)：

提示词例,可根据需求复杂设计
```markdown
# 数据分析助手

## 你的身份
你是一个专业的数据分析师，擅长Python数据处理和可视化。

## 核心能力
- 数据清洗和预处理
- 统计分析和机器学习
- 数据可视化和报告生成

## 工作流程
1. 理解数据需求
2. 探索性数据分析
3. 提供分析建议
4. 生成可执行代码
```

使用环境变量指定自定义路径：
```bash
# .env 文件
AGENT_PROJECT_PROMPT=./prompts/my_agent.md
```

### 2. 自定义工具开发

在 `project_tools/` 目录下创建Python文件，系统会自动发现并注册。

#### 工具示例

创建 `project_tools/data_tool.py`：

```python
from dbrheo.tools.base import Tool
from dbrheo.types.tool_types import ToolResult
from dbrheo.types.core_types import AbortSignal
from typing import Dict, Any, Optional

class DataAnalysisTool(Tool):
    def __init__(self, config, i18n=None):
        super().__init__(
            name="analyze_data",
            display_name="数据分析工具",
            description="对CSV数据进行统计分析",
            parameter_schema={
                "type": "object",
                "properties": {
                    "file_path": {
                        "type": "string",
                        "description": "CSV文件路径"
                    },
                    "analysis_type": {
                        "type": "string",
                        "enum": ["basic", "correlation", "distribution"],
                        "description": "分析类型"
                    }
                },
                "required": ["file_path", "analysis_type"]
            }
        )
        self.config = config
    
    def validate_tool_params(self, params: Dict[str, Any]) -> Optional[str]:
        """验证参数"""
        import os
        if not os.path.exists(params.get("file_path", "")):
            return "文件不存在"
        return None
    
    def get_description(self, params: Dict[str, Any]) -> str:
        """获取操作描述"""
        return f"分析数据文件: {params['file_path']}"
    
    async def should_confirm_execute(self, params: Dict[str, Any], signal: AbortSignal) -> bool:
        """是否需要确认"""
        return False  # 读取操作不需要确认
    
    async def execute(self, params: Dict[str, Any], signal: AbortSignal, update_output=None) -> ToolResult:
        """执行分析"""
        try:
            import pandas as pd
            
            # 读取数据
            df = pd.read_csv(params["file_path"])
            
            # 执行分析
            if params["analysis_type"] == "basic":
                result = df.describe().to_string()
            elif params["analysis_type"] == "correlation":
                result = df.corr().to_string()
            else:  # distribution
                result = f"Shape: {df.shape}\n" + df.head().to_string()
            
            return ToolResult(
                summary=f"完成{params['analysis_type']}分析",
                llm_content=result,
                return_display=f"```\n{result}\n```"
            )
        except Exception as e:
            return ToolResult(
                summary="分析失败",
                error=str(e)
            )
```

**注册要点**：
- 继承 `Tool` 基类
- 实现四个方法：`validate_tool_params`、`get_description`、`should_confirm_execute`、`execute`
- 放入 `project_tools/` 目录自动注册

### 3. 品牌定制

创建 `branding.json` 在项目根目录：

```json
{
  "name": "Your Agent",
  "ascii_art": {
    "default_array": [
      "╔══════════════════════════════╗",
      "║        YOUR AGENT            ║",
      "╚══════════════════════════════╝"
    ],
    "short": "YOUR AGENT",
    "minimal": "Agent"
  },
  "colors": {
    "gradient": ["#808080", "#A0A0A0", "#C0C0C0"],
    "tips": "#FFFFFF"
  },
  "startup_tips": [
    "输入任务开始使用",
    "使用 /help 查看命令",
    "项目工具放在 project_tools/ 目录"
  ],
  "keyboard_hints": "ESC: 清除/退出 | ''': 多行输入"
}
```

**主要配置**：
- `ascii_art.default_array` - LOGO数组（每行一个元素）
- `colors.gradient` - 渐变色数组
- `startup_tips` - 启动提示列表
- `keyboard_hints` - 键盘快捷键提示

## 内置工具

- **文件操作**：read_file、write_file、list_directory
- **代码执行**：execute_code、shell_execute
- **高性能搜索**：grep（正则搜索）、glob（文件查找）


## 主要环境变量

- `AGENT_PROJECT_PROMPT` - 项目提示词路径
- `AGENT_BRANDING_CONFIG` - 品牌配置路径
- `DBRHEO_MODEL` - 默认AI模型
- `DBRHEO_LANG` - 界面语言

## 项目结构

```
your-project/
├── PROJECT.md           # 项目提示词
├── branding.json        # 品牌配置
├── project_tools/       # 自定义工具目录
│   └── your_tool.py
├── .env                 # 环境变量
├── logs/                # 日志文件目录
├── requirements.txt     # Python依赖
├── pyproject.toml       # 项目配置
└── packages/            # 框架核心（无需修改）
```

## 🎯 使用场景示例

### 数据分析助手
```python
# 配置PROJECT.md为数据分析专家
# 然后可以直接对话：
"请分析sales_data.csv文件，生成月度销售趋势图"
"帮我清洗这个数据集，去除异常值"
"对用户行为数据进行聚类分析"
```

### 代码助手
```python
# 配置为编程助手
"请帮我重构这个Python函数，提高性能"
"生成单元测试用例"
"检查代码中的潜在bug"
```

### 文档处理
```python
# 文档自动化处理
"将这个Markdown文件转换为HTML"
"批量重命名文件夹中的图片文件"
"提取PDF中的文本内容"
```

## 🔧 高级配置

### 模型配置
```bash
# .env文件中的高级配置
DBRHEO_MODEL=gpt-4-turbo-preview
DBRHEO_TEMPERATURE=0.7
DBRHEO_MAX_TOKENS=4000
DBRHEO_LANG=zh-CN

# 代理配置（如需要）
HTTP_PROXY=http://proxy.example.com:8080
HTTPS_PROXY=https://proxy.example.com:8080
```

### 日志配置
```yaml
# log_config.yaml 自定义日志级别
version: 1
formatters:
  default:
    format: '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
handlers:
  file:
    class: logging.FileHandler
    filename: logs/agent.log
    formatter: default
    level: INFO
```

## 🐛 故障排除

### 常见问题

**Q: 启动时提示"模块未找到"错误**
```bash
# 确保在正确的虚拟环境中
source venv/bin/activate  # 或 venv\Scripts\activate
pip install -r requirements.txt
```

**Q: API调用失败**
```bash
# 检查API密钥配置
cat .env | grep API_KEY
# 检查网络连接
curl -I https://api.openai.com/v1/models
```

**Q: 自定义工具不生效**
```bash
# 检查工具文件位置和语法
python -c "import project_tools.your_tool"
# 查看日志文件
tail -f logs/agent.log
```

**Q: 内存占用过高**
```bash
# 调整模型参数
export DBRHEO_MAX_TOKENS=2000
# 或在.env中设置较小的上下文窗口
```

### 调试技巧

1. **启用详细日志**
```bash
export DBRHEO_LOG_LEVEL=DEBUG
python packages/cli/cli.py
```

2. **工具开发调试**
```python
# 在工具的execute方法中添加
import logging
logger = logging.getLogger(__name__)
logger.info(f"工具参数: {params}")
```

3. **性能监控**
```bash
# 查看资源使用情况
top -p $(pgrep -f "python packages/cli/cli.py")
```

## 📚 API参考

### 环境变量完整列表

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `DBRHEO_MODEL` | `gpt-3.5-turbo` | AI模型名称 |
| `DBRHEO_TEMPERATURE` | `0.7` | 模型创造性参数 |
| `DBRHEO_MAX_TOKENS` | `4000` | 最大token数 |
| `DBRHEO_LANG` | `zh-CN` | 界面语言 |
| `AGENT_PROJECT_PROMPT` | `PROJECT.md` | 项目提示词文件 |
| `AGENT_BRANDING_CONFIG` | `branding.json` | 品牌配置文件 |
| `OPENAI_API_KEY` | - | OpenAI API密钥 |
| `ANTHROPIC_API_KEY` | - | Claude API密钥 |

## 🤝 贡献指南

我们欢迎所有形式的贡献！

### 如何贡献

1. **Fork** 这个仓库
2. **创建** 你的特性分支 (`git checkout -b feature/AmazingFeature`)
3. **提交** 你的更改 (`git commit -m 'Add some AmazingFeature'`)
4. **推送** 到分支 (`git push origin feature/AmazingFeature`)
5. **打开** 一个Pull Request

### 开发指南

- 遵循PEP 8代码规范
- 添加适当的测试用例
- 更新相关文档
- 确保所有测试通过

### 报告问题

如果你发现了bug或有功能建议，请[创建一个Issue](https://github.com/Din829/KD0825/issues)。

## 📄 许可证

本项目采用MIT许可证 - 查看 [LICENSE](LICENSE) 文件了解详情。

## 🙏 致谢

感谢所有为这个项目做出贡献的开发者！

---

**⭐ 如果这个项目对你有帮助，请给我们一个Star！**

