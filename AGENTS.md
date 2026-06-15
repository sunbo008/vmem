# AGENTS

<skills_system priority="1">

## Available Skills

<!-- SKILLS_TABLE_START -->
<usage>
When users ask you to perform tasks, check if any of the available skills below can help complete the task more effectively. Skills provide specialized capabilities and domain knowledge.

How to use skills:
- Invoke: `npx openskills read <skill-name>` (run in your shell)
  - For multiple: `npx openskills read skill-one,skill-two`
- The skill content will load with detailed instructions on how to complete the task
- Base directory provided in output for resolving bundled resources (references/, scripts/, assets/)

Usage notes:
- Only use skills listed in <available_skills> below
- Do not invoke a skill that is already loaded in your context
- Each skill invocation is stateless
</usage>

<available_skills>

<skill>
<name>algorithmic-art</name>
<description>Creating algorithmic art using p5.js with seeded randomness and interactive parameter exploration. Use this when users request creating art using code, generative art, algorithmic art, flow fields, or particle systems. Create original algorithmic art rather than copying existing artists' work to avoid copyright violations.</description>
<location>global</location>
</skill>

<skill>
<name>brand-guidelines</name>
<description>Applies Anthropic's official brand colors and typography to any sort of artifact that may benefit from having Anthropic's look-and-feel. Use it when brand colors or style guidelines, visual formatting, or company design standards apply.</description>
<location>global</location>
</skill>

<skill>
<name>canvas-design</name>
<description>Create beautiful visual art in .png and .pdf documents using design philosophy. You should use this skill when the user asks to create a poster, piece of art, design, or other static piece. Create original visual designs, never copying existing artists' work to avoid copyright violations.</description>
<location>global</location>
</skill>

<skill>
<name>claude-api</name>
<description>|-</description>
<location>global</location>
</skill>

<skill>
<name>cpp-architect-review</name>
<description>融合Microsoft/Amazon/Oracle三大企业级架构师视角的C++审查与修复技能。覆盖架构设计、代码质量、性能优化、安全加固(SDL/STRIDE)、运维卓越(可观测性/容错)、API/ABI稳定性与向后兼容、编码规范,提供详细的改进建议和修复方案。当用户提到"审查方案"、"检查设计"、"架构评审"、"代码审查"、"C++规范检查"或需要从架构师角度评估C++代码时使用此技能。</description>
<location>global</location>
</skill>

<skill>
<name>cpp-task-driven-dev</name>
<description>C++项目迭代开发工作流。整合openspec需求分析与任务拆解、Google C++架构师角色审核、cursor-agent CLI代码开发、单测与自动化测试、验收验证的闭环迭代流程。当用户提到"任务开发"、"迭代开发"、"C++开发流程"、"C++开发"、"openspec需求"、"架构审核"、"cursor-agent开发"或需要按流程完成C++编码任务时使用此技能。</description>
<location>global</location>
</skill>

<skill>
<name>doc-coauthoring</name>
<description>Guide users through a structured workflow for co-authoring documentation. Use when user wants to write documentation, proposals, technical specs, decision docs, or similar structured content. This workflow helps users efficiently transfer context, refine content through iteration, and verify the doc works for readers. Trigger when user mentions writing docs, creating proposals, drafting specs, or similar documentation tasks.</description>
<location>global</location>
</skill>

<skill>
<name>docx</name>
<description>"Use this skill whenever the user wants to create, read, edit, or manipulate Word documents (.docx files). Triggers include: any mention of 'Word doc', 'word document', '.docx', or requests to produce professional documents with formatting like tables of contents, headings, page numbers, or letterheads. Also use when extracting or reorganizing content from .docx files, inserting or replacing images in documents, performing find-and-replace in Word files, working with tracked changes or comments, or converting content into a polished Word document. If the user asks for a 'report', 'memo', 'letter', 'template', or similar deliverable as a Word or .docx file, use this skill. Do NOT use for PDFs, spreadsheets, Google Docs, or general coding tasks unrelated to document generation."</description>
<location>global</location>
</skill>

<skill>
<name>frontend-design</name>
<description>Guidance for distinctive, intentional visual design when building new UI or reshaping an existing one. Helps with aesthetic direction, typography, and making choices that don't read as templated defaults.</description>
<location>global</location>
</skill>

<skill>
<name>fullstack-architect-review</name>
<description>以Google前后端工程架构师角色审查前后端代码。深度覆盖前端(React/Vue/TypeScript/HTML/CSS)和后端(Node.js/Python/API设计/数据库)的架构设计、代码质量、性能优化、安全性和工程规范,提供详细改进建议和修复方案。当用户提到"前端审查"、"后端审查"、"全栈审查"、"API审查"、"前后端代码审查"、"Web架构评审"、"前端性能"、"后端性能"或需要从全栈架构师角度评估代码时使用此技能。</description>
<location>global</location>
</skill>

<skill>
<name>internal-comms</name>
<description>A set of resources to help me write all kinds of internal communications, using the formats that my company likes to use. Claude should use this skill whenever asked to write some sort of internal communications (status reports, leadership updates, 3P updates, company newsletters, FAQs, incident reports, project updates, etc.).</description>
<location>global</location>
</skill>

<skill>
<name>mcp-builder</name>
<description>Guide for creating high-quality MCP (Model Context Protocol) servers that enable LLMs to interact with external services through well-designed tools. Use when building MCP servers to integrate external APIs or services, whether in Python (FastMCP) or Node/TypeScript (MCP SDK).</description>
<location>global</location>
</skill>

<skill>
<name>pdf</name>
<description>Use this skill whenever the user wants to do anything with PDF files. This includes reading or extracting text/tables from PDFs, combining or merging multiple PDFs into one, splitting PDFs apart, rotating pages, adding watermarks, creating new PDFs, filling PDF forms, encrypting/decrypting PDFs, extracting images, and OCR on scanned PDFs to make them searchable. If the user mentions a .pdf file or asks to produce one, use this skill.</description>
<location>global</location>
</skill>

<skill>
<name>pdf-password-remover</name>
<description>移除PDF文件的权限密码(Owner Password)，解锁打印、复制、编辑等受限操作。使用libqpdf C++ API实现，独立二进制无额外运行时依赖。当用户提到"PDF解密"、"移除PDF密码"、"PDF权限密码"、"解锁PDF"、"PDF打印限制"或需要移除PDF文件的操作限制时使用此技能。</description>
<location>global</location>
</skill>

<skill>
<name>pptx</name>
<description>"Use this skill any time a .pptx file is involved in any way — as input, output, or both. This includes: creating slide decks, pitch decks, or presentations; reading, parsing, or extracting text from any .pptx file (even if the extracted content will be used elsewhere, like in an email or summary); editing, modifying, or updating existing presentations; combining or splitting slide files; working with templates, layouts, speaker notes, or comments. Trigger whenever the user mentions \"deck,\" \"slides,\" \"presentation,\" or references a .pptx filename, regardless of what they plan to do with the content afterward. If a .pptx file needs to be opened, created, or touched, use this skill."</description>
<location>global</location>
</skill>

<skill>
<name>python-architect</name>
<description>以Google Python软件架构师角色进行Python软件的设计、审核、开发和验证。覆盖全生命周期：需求分析→架构设计→代码审核→实现开发→测试验证→验收归档。支持Web后端(Django/Flask/FastAPI)、数据工程/ML Pipeline、CLI工具、自动化脚本等所有Python项目类型。包含自动化质量检查脚本(ruff/mypy/bandit/pytest)。当用户提到"Python设计"、"Python审核"、"Python开发"、"Python架构"、"Python代码审查"、"Python项目"、"Python重构"、"Python测试"，或需要以架构师视角设计、审查、开发Python软件时使用此技能。即使用户只是说"写个Python脚本"或"帮我review Python代码"，只要涉及Python软件工程，都应使用此技能。</description>
<location>global</location>
</skill>

<skill>
<name>skill-creator</name>
<description>Create new skills, modify and improve existing skills, and measure skill performance. Use when users want to create a skill from scratch, edit, or optimize an existing skill, run evals to test a skill, benchmark skill performance with variance analysis, or optimize a skill's description for better triggering accuracy.</description>
<location>global</location>
</skill>

<skill>
<name>slack-gif-creator</name>
<description>Knowledge and utilities for creating animated GIFs optimized for Slack. Provides constraints, validation tools, and animation concepts. Use when users request animated GIFs for Slack like "make me a GIF of X doing Y for Slack."</description>
<location>global</location>
</skill>

<skill>
<name>stock-anomaly-analysis</name>
<description>个股异动多维度深度分析。针对某个股票，通过网络数据从游资、主力、机构、国家监管层、散户五大维度综合分析其异动（大涨、大跌、涨停、跌停、放量等）的原因，解读多方博弈格局，并预测未来发展行情。当用户提到"个股异动"、"股票为什么涨/跌"、"涨停原因"、"异动分析"、"主力分析"、"游资分析"、"机构持仓"、"多方博弈"或需要分析某只股票突然变化的原因时使用此技能。</description>
<location>global</location>
</skill>

<skill>
<name>stock-batch-scanner</name>
<description>全A股批量扫描选股。从本地K线数据或API批量筛选"近2月涨停+多头排列+右侧买点"的股票，生成交互式看板，并衔接stock-anomaly-analysis做深度分析。当用户提到"全市场扫描"、"批量选股"、"筛选股票"、"今日选股"、"扫描全A"或需要从全市场中筛选符合条件的股票时使用此技能。</description>
<location>global</location>
</skill>

<skill>
<name>template</name>
<description>Replace with description of the skill and when Claude should use it.</description>
<location>global</location>
</skill>

<skill>
<name>theme-factory</name>
<description>Toolkit for styling artifacts with a theme. These artifacts can be slides, docs, reportings, HTML landing pages, etc. There are 10 pre-set themes with colors/fonts that you can apply to any artifact that has been creating, or can generate a new theme on-the-fly.</description>
<location>global</location>
</skill>

<skill>
<name>web-artifacts-builder</name>
<description>Suite of tools for creating elaborate, multi-component claude.ai HTML artifacts using modern frontend web technologies (React, Tailwind CSS, shadcn/ui). Use for complex artifacts requiring state management, routing, or shadcn/ui components - not for simple single-file HTML/JSX artifacts.</description>
<location>global</location>
</skill>

<skill>
<name>web-search</name>
<description>Perform web searches and fetch web page content without any API keys, supporting Baidu, Bing, and DuckDuckGo search engines. Use this skill whenever the user needs to search the internet, look up current information, find documentation, research a topic, check latest news, or fetch content from a URL. Also trigger when the user mentions searching online, googling something, looking something up on the web, or needs real-time/up-to-date information that the model doesn't have. Even if the user just casually says "search for", "look up", "query", or "find", use this skill.</description>
<location>global</location>
</skill>

<skill>
<name>webapp-testing</name>
<description>Toolkit for interacting with and testing local web applications using Playwright. Supports verifying frontend functionality, debugging UI behavior, capturing browser screenshots, and viewing browser logs.</description>
<location>global</location>
</skill>

<skill>
<name>weekly-sector-heatmap</name>
<description>生成股票板块周热度跟踪图。用于收集整理本周热门股票板块TOP10，生成交互式热力图HTML文件。当用户提到"板块热度"、"热门板块"、"板块热力图"、"周板块分析"或需要分析股票市场板块趋势时使用此技能。</description>
<location>global</location>
</skill>

<skill>
<name>wps-dump-analysis</name>
<description>>-</description>
<location>global</location>
</skill>

<skill>
<name>xlsx</name>
<description>"Use this skill any time a spreadsheet file is the primary input or output. This means any task where the user wants to: open, read, edit, or fix an existing .xlsx, .xlsm, .csv, or .tsv file (e.g., adding columns, computing formulas, formatting, charting, cleaning messy data); create a new spreadsheet from scratch or from other data sources; or convert between tabular file formats. Trigger especially when the user references a spreadsheet file by name or path — even casually (like \"the xlsx in my downloads\") — and wants something done to it or produced from it. Also trigger for cleaning or restructuring messy tabular data files (malformed rows, misplaced headers, junk data) into proper spreadsheets. The deliverable must be a spreadsheet file. Do NOT trigger when the primary deliverable is a Word document, HTML report, standalone Python script, database pipeline, or Google Sheets API integration, even if tabular data is involved."</description>
<location>global</location>
</skill>

</available_skills>
<!-- SKILLS_TABLE_END -->

</skills_system>
