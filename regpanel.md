<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<base target="_blank" />

# 寄存器面板：一款自制的寄存器信息转换程序

## 1、背景

* 不论做`单片`机开发，还是与硬件相关的`Linux`驱动开发，**很大一部分的关注点都是配置寄存器**。

* 要正确地配置一个寄存器，就要先搞清楚该寄存器的用途、时序要求以及它里面**每一个二进制位的含义**。
对于最后一个要求，常见的做法是查阅芯片的数据手册，并将取得的**目标寄存器的当前值拆分成多个部分**，
再将**每部分的数值**逐个与手册对照并**翻译成**具体的**业务含义**。

* 显然，这种翻译虽然能让懒散的大脑做一下“保健操”，但这种工作毫无技术含量可言，
而且**次数一多，也是相当乏味**，**尤其对于**一个**懒人**来说**更是无异于反复处刑**。

* 当然，除了调试场景，对于更加日常化的开发场景，可以通过注释的方式在预设值附近解释此值的含义，
而对于程序运行期打印的值则可以编写相应的解析函数来转换，但这无疑会增加注释量和代码量，
对代码的可维护性造成威胁，为向`屎山`进发的道路上加了一脚油门，即使用脚趾来思考都知道并不可取。
不过，对于将代码量作为一项绩效的公司来说，却是冲`KPI`的好理由。

* 抛开对有毒`KPI`的追求和虚假勤奋带来的自我感动，若从**维持核心业务代码的精简性及稳定性**、
**发掘一套**不限于特定的单一的项目才适用的**通用做法**的角度来考虑，
要是能**有一个小工具可接过这种翻译工作**，那么在**核心业务代码里**只需**简单地按照一定格式打印出那些寄存器数值**，
再将这些数值**复制给小工具**进行**批量翻译**（甚至还能借助脚本来处理），
更重要的是不必再在不同项目里针对一个个具体的寄存器一次又一次地写注释或解析函数，
这无疑是**对开发人员精力的极大解放**。

## 2、预期目标及设计思路

* 经过前面的背景介绍，很容易对目标小工具有一些初步的构想，如下：
    * 必须支持**图形化界面（即`GUI`）操作**，因为字符界面不好呈现，也不方便操作
    ——在脑海里想象一下便会明白。
    * 必须支持**不限定数量及类型的芯片**，并且**允许用户对数据进行增、删、改**，即**支持定制**<a id="customized_req"></a>。
    * 操作**界面**必须设计得**简洁、高效**：核心信息要一眼即见，且不可有所缺漏；
    补充信息不能喧宾夺主，也不能过多，以免造成界面拥护，但也要留有查询渠道。
    * 作为前一点要求的延伸，根据实际操作的需求，**部分信息的展示**要设成**自动触发**，
    而**另一部分**则是**手动触发**。
    * 既要**定好输入**形式（例如命令行参数、特定格式的配置文件等），又要**定好输出**形式（例如表格、
    文本框等），**有关联的控件还必须能联动变化**，且每一组/个**二进制位**必须**体现出读写权限**。
    * 最后，界面的**外观不作要求**，因为供技术人员使用，实用性至上，
    越花哨的外观反而带来越重的兼容性难度和维护包袱。按本人一贯的风格，
    为求省事干脆将其做成简单的**暗色调**，因为**不刺眼**。

* 上面既已明确了功能需求，接下来再梳理技术方案也并非难事。要做好这一阶段，
其实有法可依，同时又有很大自由发挥空间，现实中更多是依赖作者的已有经验和个人偏好。
由于篇幅所限，决策过程中的一些不重要细节就不过多展开，仅论述必要的重点。
首先是对最基础的通用技术的考虑：
    * 决定目标小工具的**呈现形式**：**桌面电脑客户端程序**。与此相对的是网页和手机应用程序
    ——之所以不考虑这两个，是因为网页版会受制于浏览器的安全策略而不便访问任意的目录和文件，
    而且做不了版本管理（指的是将某一次`Git`或`SVN`提交形成的版本号固化到目标程序里）；
    至于手机应用程序就更不用说，巴掌大的屏幕费眼，触摸操作费手指且低效，
    与该工具的应用场景极度不协调，根本不值得花时间考虑。
    * 决定**受支持平台**（操作系统）：`Linux`。无他，只因作者专注于`Linux`开发，
    主力系统早已切换成`Linux`，`Windows`在日常中几乎用不到，而用不到的系统自然会带来维护上的不便，
    所以只要是个人项目，基本上都优先支持甚至仅支持`Linux`，此项目也不例外。
    * 决定**编程语言**和基础**库/框架**：综观当前主流语言的特性及其在`GUI`编程的生态，
    决定选用`C++`（这也是作者日常使用的其中一种语言）。至于`GUI`库的第一反应则是`GTK`或`Qt`，
    再考虑多几秒便决定选用`Qt`——因为实测`Qt`的开发效率（不是程序运行效率）确实非常高，
    归功于其`API`设计得很合理，面向对象的理念运用得恰到好处，
    还自创`元对象编译器`（`moc`: `Meta Object Compiler`）扩展了`C++`标准以外的特性，
    并且其配套设计工具（`Qt Designer`，译名：`Qt设计师/器`）用起来非常方便，
    文档也写得不错（`Qt Assistant`，译名：`Qt助手`），不失为一个极好的学习案例和非商业项目的开发利器。
    * 最后但最重要的是必须支持**配置文件**：除了前面提到的<a href="#customized_req" target="_self">可定制</a>要求外，
    还因为对于此类项目来说，有了配置文件便等于有了**数据积累**——不论做技术还是做业务，
    做到最后一定是在做数据，多观察几个现实案例和当前环境显现出来的趋势（尤其是人工智能领域）便明白。

* 明确了通用类的技术方案，再搞掂与具体业务相关的专用技术点便差不多可以收工了：
    * **寄存器**的各个二进制位**信息很规整**，**最适合使用表格**来展示。
    `Qt`有现成的`QTable*`类可供使用。
    * **数字**类信息可使用专门的输入框，省却自己动手书写格式限制和数值校验的逻辑。
    `Qt`有现成的`SpinBox`类可供使用，但不完全满足要求，需要扩展一下，后文会解释。
    * **枚举**类信息可使用**下拉框**来展示。`Qt`有现成的`QComboBox`类可供使用。
    * 对于名称、标题等较短的**核心信息**，**可以直接展示**；至于**补充信息或冗长的内容**，
    可以**不直接显示**，而是**给定一个关联标签**并为其添加下划线以便突出显示，且**光标移过去时**，
    会通过**悬浮提示**的形式**将详细内容显示出来**，这样便能既兼顾了界面布局，
    又保证了信息的完整性。
    * 最后是**配置文件的选型**：还是参考业界常用的方案，主要候选者有`XML`、`JSON`和`YAML`。
    `XML`书写起来太冗长，不选；`JSON`的格式看起来非常舒服，用过的都说好（除了不支持注释稍让人不爽），
    而且`Qt`也有读写`JSON`的`API`，不需要另外安装第三方库，所以就**决定是`JSON`了**；
    `YAML`也不错，且含有比`JSON`更强大更灵活的扩展语法，但个人不喜欢其将缩进作为一项语法规则的做法，
    故不采用。
    * 最后的最后，还要思考**配置内容的规范**：在满足`JSON`基本语法的前提下，
    不同业务对配置文件的内容显然有不同要求，姑且将这些要求称之为`微语法`或`业务格式`。
    对于本项目而言，需要考虑一个寄存器配置项有哪些必要字段、有哪些可选字段、
    当字段定义与另一个寄存器完全相同时是否支持直接引用以便大幅减少配置内容等等。

* 最后是代码思路。经过前面的分析，可知整个项目的基石就是`Qt`，
所以编程思路就简化为遵循`Qt`的设计范式、用好它提供的资源就行了。当然，
在个别地方会有一些陷阱，详见下文。

## 3、若干障碍列举

* `Qt Designer`与`元对象编译器`（`moc`）存在部分不一致的行为：
例如在`Designer`里修改了某个控件的底色，但经过`moc`生成代码并编译出目标程序之后，
会发现底色并未变化，需要在用户代码里手动调用`setStyleSheet()`，
通过修改`层叠样式表`（`Cascading Style Sheets`，即`CSS`）的方式来调整底色，
但这种方法又要注意`调色板`（`QPalette`）与`CSS`的优先级问题，即在代码里的调用顺序。

* 表格控件既可使用`QTableView`，也可使用`QTableWidget`，但前者默认只支持文字，
后者更灵活，但子控件的数目和嵌套层数非常多，调用`dumpObjectTree()`打印一个表格控件稍微感受一下：
    ````
    QTableWidget::reg[1]_holder                             # 用户定义的最外层表格容器
        QWidget::qt_scrollarea_viewport
            QLineEdit::reg[1]_title                         # 寄存器标题
                QWidgetLineControl::
            QTableWidget::reg[1]_full_values                # 存放寄存器完整的数值的表格
                QWidget::qt_scrollarea_viewport
                    QLabel::reg[1]_full_values_def_label    # 默认值标签
                    U64SpinBox::reg[1]_full_values_def_val  # 完整的默认值
                        QLineEdit::qt_spinbox_lineedit
                            QWidgetLineControl::
                        QValidator::qt_spinboxvalidator
                    QLabel::reg[1]_full_values_curr_label   # 当前值标签
                    U64SpinBox::reg[1]_full_values_curr_val # 完整的当前值
                        QLineEdit::qt_spinbox_lineedit
                            QWidgetLineControl::
                        QValidator::qt_spinboxvalidator
                QWidget::qt_scrollarea_vcontainer
                    QScrollBar::
                    QBoxLayout::
                QStyledItemDelegate::
                QHeaderView::
                    QWidget::qt_scrollarea_viewport
                    QWidget::qt_scrollarea_hcontainer
                        QScrollBar::
                        QBoxLayout::
                    QWidget::qt_scrollarea_vcontainer
                        QScrollBar::
                        QBoxLayout::
                    QItemSelectionModel::
                QHeaderView::
                    QWidget::qt_scrollarea_viewport
                    QWidget::qt_scrollarea_hcontainer
                        QScrollBar::
                        QBoxLayout::
                    QWidget::qt_scrollarea_vcontainer
                        QScrollBar::
                        QBoxLayout::
                    QItemSelectionModel::
                QTableCornerButton::
                QTableModel::
                QItemSelectionModel::
                QWidget::qt_scrollarea_hcontainer
                    QScrollBar::
                    QBoxLayout::
            RegBitsTable::reg[1]_bits                       # 二进制位详情表格（扩展过的表格子类）
                QWidget::qt_scrollarea_viewport
                    QLabel::reg[1]_bits[31:1]               # 其中一组二进制位的标签
                    U64SpinBox::reg[1]_bits[31:1]_defval    # 该组二进制位的默认值（片段）
                        QLineEdit::qt_spinbox_lineedit
                            QWidgetLineControl::
                        QValidator::qt_spinboxvalidator
                    U64SpinBox::reg[1]_bits[31:1]_currval   # 该组二进制位的当前值（片段）
                        QLineEdit::qt_spinbox_lineedit
                            QWidgetLineControl::
                        QValidator::qt_spinboxvalidator
                    QLabel::reg[1]_bits[31:1]_desc          # 该组二进制位的描述，此例只有简单标签文字
    # 后面省略100多行
    ````
    可以看出实际生成的对象（未用注释标注出来的那些）数目是非常多的，
    而且它们之间的上下级关系数目和嵌套层数也很多，尽管部分对象是由类的继承和组合而产生的，
    但在实际的编程中还是要非常小心，不要搞错层次关系，否则很容易找错对象，
    导致类型转换错误而产生空指针，进而导致程序崩溃。而且，也不能假设以上的模型是固定的，
    因为后续版本的`Qt`源码实现可能会变化，或者`moc`生成的代码有变化。基于这些考虑，
    由用户控制的控件应该按一定规范进行命名，然后在各层循环遍历时按名称查找，这样才最保险。
    详细逻辑就不在此展开，感兴趣者可访问文末的项目链接去查阅。

* 关于表格控件，还有一个陷阱，就是其默认参数生成的界面效果的自适应性非常差，
父子控件之间很容易发生遮挡，往往需要手动调整。

* 数值框`SpinBox`类的`32`位**带符号**整数（`int32`）范围限制。
这意味着能输入的最大值是`0x7FFFFFFF`，对于`32`位寄存器最高位允许取`1`的情况是不支持的，
更不要说想支持`64`位寄存器了。要解决这个问题，只能重新造一个轮子，
或者继承`SpinBox`（或其父类`QAbstractSpinBox`）并重新实现必要的成员函数——
本项目采用的是后一种做法。通过查阅`Qt`源码，得知至少需要重新实现`3`个虚函数：`validate()`、
`textFromValue()`和`valueFromText()`，但由于最后一个`valueFromText()`的返回值`int`类型
无法更改成`uint64_t`（由于`C++`的语法限制），必须另想他法，只能实现前`2`个。
并且，重新实现`textFromValue()`带来一些副作用，令方框的上下箭头不能正常工作，
不得不重新实现多一个虚拟函数`stepBy()`。外加`uint64_t`类型必需的其他逻辑，核心改动如下：
    ````
    /**************** 头文件 ****************/
    class U64SpinBox : public QSpinBox
    {
        Q_OBJECT

        // ...

        inline uint64_t value(void) const
        {
            return m_value64;
        }

        inline uint64_t minimum(void) const
        {
            return m_minimum64;
        }

        inline void setMinimum(uint64_t min)
        {
            if (min <= m_maximum64)
            {
                int64_t signed_min = static_cast<int64_t>(min);

                m_minimum64 = min;
                // For automatically disabling the Down button of combo box if reaching the bottom.
                QSpinBox::setMinimum((signed_min < INT32_MIN) ? INT32_MIN : signed_min);
            }
        }

        inline uint64_t maximum(void) const
        {
            return m_maximum64;
        }

        inline void setMaximum(uint64_t max)
        {
            if (max >= m_minimum64)
            {
                m_maximum64 = max;
                // For automatically disabling the Up button of combo box if reaching the top.
                QSpinBox::setMaximum((max > INT32_MAX) ? INT32_MAX : max);
            }
        }

        inline void setRange(uint64_t min, uint64_t max)
        {
            if (max >= min)
            {
                setMinimum(min);
                setMaximum(max);
            }
        }

    protected:
        QValidator::State validate(QString &input, int &pos) const override;
        //int valueFromText(const QString &text) const override;
        QString textFromValue(int val) const override;

    public:
        void stepBy(int steps) override;

    public Q_SLOTS:
        void setValue(uint64_t val);

    Q_SIGNALS:
      void valueChanged(uint64_t);

    private:
        uint64_t m_value64;
        uint64_t m_minimum64;
        uint64_t m_maximum64;
    };

    /**************** 源文件 ****************/

    QValidator::State U64SpinBox::validate(QString &input, int &pos) const/* override */
    {
        QString copy(input);

        if (copy.startsWith("0x"))
            copy.remove(0, 2);

        pos -= copy.size() - copy.trimmed().size();
        copy = copy.trimmed();
        if (copy.isEmpty())
            return QValidator::Intermediate;

        const std::string copy_str = copy.toStdString();
        char *end_ptr;
        uint64_t val = strtoull(copy_str.c_str(), &end_ptr, this->displayIntegerBase());

        if (/*'\0' != copy_str[0] && */'\0' == *end_ptr) // Entire string is valid.
            return (val >= m_minimum64 && val <= m_maximum64) ? QValidator::Acceptable : QValidator::Invalid;

        return QValidator::Invalid;
    }

    QString U64SpinBox::textFromValue(int val/* This value is truncated and thus not used. */) const
    {
        const QString &text = this->lineEdit()->displayText();
        int base = this->displayIntegerBase();
        uint64_t value = text.toULongLong(nullptr, base); // Converted from instant text instead of using the old m_value64.

        return QString::number(value, base);
    }

    void U64SpinBox::stepBy(int steps)
    {
        if (0 == steps)
            return;

        uint64_t val = this->lineEdit()->displayText().toULongLong(nullptr, this->displayIntegerBase()); // this->value()
        uint64_t min = this->minimum();
        uint64_t max = this->maximum();

        if ((steps < 0 && val <= min) || (steps > 0 && val >= max))
            return;

        uint64_t absolute_val = (steps < 0) ? -steps : steps;

        if (steps > 0)
            this->setValue((max - val >= absolute_val) ? (val + absolute_val) : max);
        else
            this->setValue((val - min >= absolute_val) ? (val - absolute_val) : min);
    }

    void U64SpinBox::setValue(uint64_t val)
    {
        if (/*val == m_value64 || */val < m_minimum64 || val > m_maximum64)
            return;

        QString text = this->prefix() + QString::number(val, this->displayIntegerBase());

        m_value64 = val;
        this->lineEdit()->setText(text);

        //emit this->valueChanged(val);
        emit this->textChanged(this->lineEdit()->displayText());
    }
    ````
    当然，以上代码远不是最终版，因为只支持无符号整数（最大`64`位），
    这虽能应付目前遇到的绝大多数寄存器的字段定义，但只有同时也支持带符号整数，
    才算功德圆满。

* **注**：以上列举的并非全部的重点和难点，之所以只列举这些，是因为它们的出现有悖于常规的直觉，
属于超出正常预期的意外。若有兴趣了解其他的重点难点甚至整个项目的代码实现，可访问文末的项目链接去查阅。
并且，本文重在解释思路，而非精确的实现细节，因此项目代码后续所有的更新都不会再同步反映到本文。

## 4、项目链接

* [转换程序](https://github.com/FooFooDamon/regpanel)

* [配置文件](https://github.com/FooFooDamon/regpanel-conf)

