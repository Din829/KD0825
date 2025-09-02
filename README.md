# 通用Agent框架使用指南

本框架是一个完全泛用化的Agent开发平台，提供了核心能力和定制接口。

## 快速开始

```bash
# 安装依赖
pip install -r requirements.txt

# 复制环境文件并配置APIkey
cp .env.example .env

# 启动Agent
python packages/cli/cli.py
```

*注意，不推荐使用Claude模型，因为没有做提示词缓存，消耗量太大。

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

## 常见问题 FAQ

### Q: 如何调试自定义工具？
A: 可以在工具的execute方法中添加日志输出，日志会保存在logs目录中。

### Q: 支持哪些AI模型？
A: 支持OpenAI GPT系列、Claude系列等主流模型，通过DBRHEO_MODEL环境变量配置。

### Q: 如何处理大文件？
A: 建议使用内置的文件操作工具，支持流式处理和分块读取。

## 贡献指南

欢迎提交Issue和Pull Request来改进这个项目！

