### 通用报表模板布局

|       元素       | 描述                                                         |
| :--------------: | ------------------------------------------------------------ |
|    **TTITLE**    | 报告的标题。只出现在报告第一页的最开头。                     |
|  **PAGEHEADER**  | 可能包含日期时间/组织结构等信息。出现在每页的顶部。          |
| **COLUMNHEADER** | 列出在报告中显示的特定字段的名称。如作者姓名、开始时间、完成时间等摘要信息 |
|    **DETAIL**    | 报告正文部分                                                 |
| **COLUMNFOOTER** | 可以显示任何字段的总和                                       |
| **PAGERFOOTER**  | 可能包含每页的信息。出现在每个页面的底部，比如“1/23”。       |
|   **SUMMARY**    | 摘要包含从“详细”部分推断出的信息。例如，列出每个作者工作的小时数后，每个作者的数量列表，总工时为每个座位可以把视力表像饼图，曲线图等，为更好的比较。 |

### JasperReport特点

* 丰富的报表导出格式。比如HTML、文本、PDF、EXCEL、WORD、RTF、PDF、XML或图像。
* 容易复杂的报表功能。比如子报表、交叉表报告、图标报表（图、饼图、XY折线图、条形图、仪表等）
* 其他特点：布局灵活、生成水印、支持多种多个数据源。

### JasperReport工具库设计概览

有很多类，用于编译`jrxml`报表设计，填充报表、打印报表、导出为PDF、HTML和XML文件，查看生成的报表和报表设计。

<img src="/Users/tanglongan/Notes/报表工具/.images//image-20201029102257809.png" alt="image-20201029102257809" style="zoom:67%;" />

* JasperCompileManager：用于编译jrxml报表模板。
* JasperFillManager：用于填充一个报表，从数据源的数据。
* JasperPrintManager：用于打印JasperReport类库生成的文件。
* JasperExportManager：用于获取PDF、HTML或XML内容以供报表填充过程中产生的文件。
* JasperViewer：一个简单的Java Swing应用程序，可以加载和显示报表。
* JasperDesignViewer：用于设计时预览报表模板。













