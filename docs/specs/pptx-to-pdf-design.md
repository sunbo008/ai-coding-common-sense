# PPTX 转 PDF 工具设计文档

> **状态**：待审核  
> **创建日期**：2026-02-03  
> **作者**：[开发者]  
> **审核人**：[架构师]

---

## 1. 概述

### 1.1 背景和目标

开发一个命令行工具，将 PowerPoint 文件（.pptx）转换为 PDF 文件。转换过程分为两步：
1. 将 PPTX 的每一页转换为图片
2. 将所有图片合成为单个 PDF 文件

### 1.2 问题域描述

- 用户有大量 PPTX 文件需要转换为 PDF 格式
- 需要保持幻灯片的视觉效果
- 需要支持批量处理

---

## 2. 需求分析

### 2.1 功能需求

| 编号 | 需求 | 优先级 |
|------|------|--------|
| FR-01 | 支持将单个 PPTX 文件转换为 PDF | P0 |
| FR-02 | 支持指定输出文件路径 | P0 |
| FR-03 | 支持自定义图片分辨率/DPI | P1 |
| FR-04 | 支持批量处理多个 PPTX 文件 | P1 |
| FR-05 | 支持指定转换的页面范围 | P2 |
| FR-06 | 显示转换进度 | P2 |

### 2.2 非功能需求

| 编号 | 需求 | 指标 |
|------|------|------|
| NFR-01 | 转换速度 | 10 页 PPTX < 10 秒 |
| NFR-02 | 内存占用 | 处理 100 页 PPTX 时 < 500MB |
| NFR-03 | 图片质量 | 默认 DPI >= 150 |
| NFR-04 | 跨平台 | 支持 Windows、macOS、Linux |

### 2.3 约束条件

- 使用 Python 实现（便于跨平台）
- 优先使用开源库，避免依赖 Microsoft Office
- 命令行界面，便于集成到自动化流程

---

## 3. 接口设计

### 3.1 命令行接口

```bash
# 基本用法
pptx2pdf input.pptx -o output.pdf

# 指定 DPI
pptx2pdf input.pptx -o output.pdf --dpi 300

# 批量处理
pptx2pdf *.pptx -o ./output/

# 指定页面范围
pptx2pdf input.pptx -o output.pdf --pages 1-5,8,10

# 显示帮助
pptx2pdf --help
```

### 3.2 Python API

```python
from pptx2pdf import PPTXConverter

# 基本用法
converter = PPTXConverter()
converter.convert("input.pptx", "output.pdf")

# 高级用法
converter = PPTXConverter(dpi=300, temp_dir="/tmp/pptx2pdf")
converter.convert(
    input_path="input.pptx",
    output_path="output.pdf",
    pages=[1, 2, 3, 5],  # 指定页面
    progress_callback=lambda current, total: print(f"{current}/{total}")
)
```

### 3.3 核心类定义

```python
class PPTXConverter:
    """PPTX 转 PDF 转换器"""
    
    def __init__(self, dpi: int = 150, temp_dir: str | None = None):
        """
        初始化转换器
        
        Args:
            dpi: 图片分辨率，默认 150
            temp_dir: 临时文件目录，默认使用系统临时目录
        """
        pass
    
    def convert(
        self,
        input_path: str | Path,
        output_path: str | Path,
        pages: list[int] | None = None,
        progress_callback: Callable[[int, int], None] | None = None
    ) -> Path:
        """
        将 PPTX 转换为 PDF
        
        Args:
            input_path: 输入 PPTX 文件路径
            output_path: 输出 PDF 文件路径
            pages: 要转换的页面列表（1-indexed），None 表示全部
            progress_callback: 进度回调函数，参数为 (当前页, 总页数)
        
        Returns:
            输出 PDF 文件的路径
        
        Raises:
            FileNotFoundError: 输入文件不存在
            ValueError: 页面范围无效
            ConversionError: 转换过程出错
        """
        pass
    
    def pptx_to_images(
        self,
        input_path: str | Path,
        output_dir: str | Path,
        pages: list[int] | None = None
    ) -> list[Path]:
        """
        将 PPTX 转换为图片序列
        
        Returns:
            生成的图片文件路径列表（按页面顺序）
        """
        pass
    
    def images_to_pdf(
        self,
        image_paths: list[Path],
        output_path: str | Path
    ) -> Path:
        """
        将图片序列合成 PDF
        
        Returns:
            输出 PDF 文件的路径
        """
        pass
```

---

## 4. 详细设计

### 4.1 技术选型

| 功能 | 推荐库 | 备选方案 | 说明 |
|------|--------|----------|------|
| PPTX 解析 | python-pptx | - | 纯 Python，无需 Office |
| 幻灯片渲染 | pdf2image + LibreOffice | unoconv | 需要 LibreOffice |
| 图片处理 | Pillow | opencv-python | 轻量级 |
| PDF 生成 | reportlab 或 img2pdf | PyPDF2 | img2pdf 更简单 |
| CLI 框架 | click 或 typer | argparse | 更友好的 API |

**注意**：纯 Python 方案（python-pptx）无法完美渲染复杂动画和特殊字体。如需高保真转换，建议使用 LibreOffice 后端。

### 4.2 架构图

```
┌─────────────────────────────────────────────────────────┐
│                     pptx2pdf CLI                        │
│                    (click/typer)                        │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   PPTXConverter                         │
│  ┌─────────────────┐    ┌─────────────────────────┐    │
│  │ pptx_to_images()│───▶│    images_to_pdf()      │    │
│  └─────────────────┘    └─────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
          │                           │
          ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│  渲染后端        │         │  PDF 生成       │
│  - LibreOffice  │         │  - img2pdf      │
│  - python-pptx  │         │  - reportlab    │
└─────────────────┘         └─────────────────┘
```

### 4.3 转换流程

```
1. 验证输入文件存在且为有效 PPTX
       ↓
2. 创建临时目录存放中间图片
       ↓
3. 调用渲染后端将 PPTX 转为图片
   - LibreOffice: soffice --headless --convert-to png
   - python-pptx: 逐页渲染（功能有限）
       ↓
4. 按页面顺序收集图片文件
       ↓
5. 使用 img2pdf 或 Pillow 合成 PDF
       ↓
6. 清理临时文件
       ↓
7. 返回输出文件路径
```

### 4.4 错误处理

| 错误类型 | 处理方式 |
|----------|----------|
| 文件不存在 | 抛出 FileNotFoundError，提示文件路径 |
| 非 PPTX 格式 | 抛出 ValueError，提示支持的格式 |
| LibreOffice 未安装 | 提示安装命令，降级使用纯 Python 方案 |
| 转换失败 | 抛出 ConversionError，包含详细错误信息 |
| 磁盘空间不足 | 检测可用空间，提前警告 |

---

## 5. 测试策略

### 5.1 行为规格（可直接转化为测试用例）

#### 基本功能测试

| 编号 | 测试场景 | 输入 | 预期输出 |
|------|----------|------|----------|
| TC-01 | 转换单页 PPTX | 1 页的 PPTX | 1 页的 PDF |
| TC-02 | 转换多页 PPTX | 10 页的 PPTX | 10 页的 PDF |
| TC-03 | 指定输出路径 | input.pptx, -o /tmp/out.pdf | PDF 生成在指定路径 |
| TC-04 | 指定 DPI | --dpi 300 | 图片分辨率为 300 DPI |
| TC-05 | 指定页面范围 | --pages 1-3 | PDF 只包含前 3 页 |

#### 边界条件测试

| 编号 | 测试场景 | 输入 | 预期输出 |
|------|----------|------|----------|
| TC-10 | 空 PPTX | 0 页的 PPTX | 抛出 ValueError |
| TC-11 | 超大 PPTX | 500 页的 PPTX | 正常转换，内存 < 1GB |
| TC-12 | 文件不存在 | nonexistent.pptx | 抛出 FileNotFoundError |
| TC-13 | 非 PPTX 文件 | document.docx | 抛出 ValueError |
| TC-14 | 无效页面范围 | --pages 100 (只有 10 页) | 抛出 ValueError |
| TC-15 | 输出目录不存在 | -o /nonexistent/out.pdf | 自动创建目录或抛出错误 |

#### 内容测试

| 编号 | 测试场景 | 验证点 |
|------|----------|--------|
| TC-20 | 包含文字的 PPTX | 文字清晰可读 |
| TC-21 | 包含图片的 PPTX | 图片完整保留 |
| TC-22 | 包含图表的 PPTX | 图表正确渲染 |
| TC-23 | 包含中文的 PPTX | 中文字体正确显示 |
| TC-24 | 包含特殊字体的 PPTX | 字体回退处理 |

### 5.2 性能测试

| 编号 | 测试场景 | 指标 |
|------|----------|------|
| PT-01 | 10 页 PPTX 转换时间 | < 10 秒 |
| PT-02 | 100 页 PPTX 内存占用 | < 500MB |
| PT-03 | 并发转换 5 个文件 | 无崩溃，正确完成 |

### 5.3 测试数据准备

需要准备以下测试用 PPTX 文件：
- `test_1page.pptx` - 单页简单内容
- `test_10pages.pptx` - 10 页混合内容
- `test_chinese.pptx` - 包含中文
- `test_images.pptx` - 包含大量图片
- `test_charts.pptx` - 包含图表
- `test_empty.pptx` - 空白 PPTX
- `test_500pages.pptx` - 压力测试用

---

## 6. 风险评估

### 6.1 已知风险及缓解措施

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| LibreOffice 依赖 | 跨平台部署复杂 | 提供 Docker 镜像，文档说明安装方式 |
| 特殊字体缺失 | 渲染效果差 | 嵌入常用字体，提供字体映射配置 |
| 大文件内存溢出 | 程序崩溃 | 分页处理，流式写入 PDF |
| 复杂动画无法渲染 | 内容丢失 | 文档说明限制，只渲染静态帧 |

### 6.2 兼容性影响

- Windows：需安装 LibreOffice（推荐 Portable 版本）
- macOS：`brew install libreoffice`
- Linux：`apt install libreoffice`

---

## 7. 项目结构

```
pptx2pdf/
├── pptx2pdf/
│   ├── __init__.py
│   ├── converter.py      # 核心转换器
│   ├── backends/
│   │   ├── __init__.py
│   │   ├── libreoffice.py  # LibreOffice 后端
│   │   └── pptx_native.py  # 纯 Python 后端
│   ├── pdf_builder.py    # PDF 生成
│   ├── cli.py            # 命令行入口
│   └── exceptions.py     # 自定义异常
├── tests/
│   ├── __init__.py
│   ├── test_converter.py
│   ├── test_backends.py
│   ├── test_cli.py
│   └── fixtures/         # 测试用 PPTX 文件
├── pyproject.toml
├── README.md
└── LICENSE
```

---

## 8. 里程碑计划

| 阶段 | 内容 | 验收标准 |
|------|------|----------|
| M1 | 基础框架 + LibreOffice 后端 | TC-01, TC-02, TC-03 通过 |
| M2 | CLI 完善 + DPI/页面范围支持 | TC-04, TC-05 通过 |
| M3 | 错误处理 + 边界条件 | TC-10 ~ TC-15 通过 |
| M4 | 性能优化 + 文档 | PT-01, PT-02 通过 |

---

## 审核记录

| 日期 | 审核人 | 结果 | 备注 |
|------|--------|------|------|
| | | | |
