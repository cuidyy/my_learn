# JSON学习

### JSON(JavaScript Object Notation)是一种轻量级的数据交换格式，采用完全独立于编程语言的文本格式来存储和表示数据，常用于在客户端和服务器之间传输数据。

**JsonCpp**是一个开源的C++库，用于解析和生成JSON（JavaScript Object Notation）数据格式。JSON是一种轻量级的数据交换格式，广泛用于各种应用程序和网络服务中。JsonCpp提供了简单和易用的API，可以方便地解析JSON字符串并将其转换为C++对象，同时也可以将C++对象序列化为JSON字符串

主要功能
1. 解析JSON字符串：JsonCpp可以将JSON格式的字符串解析成C++中的对象（通常是Json::Value类型的对象），以便在程序中进行操作。
2. 生成JSON字符串：JsonCpp也可以将C++对象（特别是Json::Value类型的对象）序列化为JSON格式的字符串，便于数据的存储和传输。
3. 操作JSON数据：JsonCpp提供了丰富的API来操作JSON数据，包括访问和修改JSON对象的属性、数组操作、遍历等。

JsonCpp主要包含以下几个基础类：
 
1. Json::Value：用于存储键值对。Json::Value是JsonCpp中最重要的类，它代表了一个JSON值，这个值可以是字符串、数字、布尔值、null、数组或对象。
2. Json::Reader：用于读取JSON文档，或者说是用于将字符串或者文件输入流转换为Json::Value对象。
3. Json::Writer：负责将内存中的Json::Value对象转换为JSON文档，输出到文件或者字符串中。Json::Writer有两种主要的方法：Fas tWriter和StyledWriter。FastWriter快速无格式地将Json::Value转换成JSON文档，而StyledWriter则有格式地将Json::Value转换成JSON文档。

## Value类
Json::Value是JSON数据的通用类型。可以表示JSON对象，数组，字符串，数字，布尔值和null等各种类型。
```c++
class Json::Value{
    //Value重载了[]和=，因此所有的赋值和获取数据都可以通过
    Value &operator=(const Value &other); 
    Value& operator[](const std::string& key);//示例:val["姓名"] = "小明";
    Value& operator[](const char* key);
    //移除元素
    Value removeMember(const char* key);
 
    //访问数组元素
    const Value& operator[](ArrayIndex index) const; //示例:val["成绩"][0]
    //添加数组元素
    Value& append(const Value& value);//示例:val["成绩"].append(88); 
    //获取数组元素个数
    ArrayIndex size() const;//示例:val["成绩"].size();
 
    //访问数据，以string类型返回
    std::string asString() const;//示例:string name = val["name"].asString();
    //访问数据，以const char*类型返回
    const char* asCString() const;//示例:char *name = val["name"].asCString();
    //访问数据，以int类型返回
    Int asInt() const;//示例:int age = val["age"].asInt();
    //访问数据，以float类型返回
    float asFloat() const;
    //访问数据，以bool类型返回
    bool asBool() const;
};
```
**构造函数**
```c++
Value(ValueType type = nullValue);
Value(Int value);
Value(UInt value);
Value(Int64 value);
Value(UInt64 value);
Value(double value);
Value(const char* value);
Value(const char* begin, const char* end);
Value(bool value);
Value(const Value& other);
Value(Value&& other);
```

**检测数据类型**
```c++
bool isNull() const;
bool isBool() const;
bool isInt() const;
bool isInt64() const;
bool isUInt() const;
bool isUInt64() const;
bool isIntegral() const;
bool isDouble() const;
bool isNumeric() const;
bool isString() const;
bool isArray() const;
bool isObject() const;
```
**将Value对象转换为实际类型**
```c++
Int asInt() const;
UInt asUInt() const;
Int64 asInt64() const;
UInt64 asUInt64() const;
LargestInt asLargestInt() const;
LargestUInt asLargestUInt() const;
JSONCPP_STRING asString() const;
float asFloat() const;
double asDouble() const;
bool asBool() const;
const char* asCString() const;
```

**将Value对象数据序列化为string**
```c++
// 序列化得到的字符串有样式 -> 带换行 -> 方便阅读
// 写配置文件的时候
std::string toStyledString() const;
```

## FastWriter类
```c++
class JSON_API FastWriter
    : public Writer {
public:
  FastWriter();
  ~FastWriter() override = default;

  void enableYAMLCompatibility();

  /** \brief Drop the "null" string from the writer's output for nullValues.
   * Strictly speaking, this is not valid JSON. But when the output is being
   * fed to a browser's JavaScript, it makes for smaller output and the
   * browser can handle the output just fine.
   */
  void dropNullPlaceholders();

  void omitEndingLineFeed();

public: // overridden from Writer
  String write(const Value& root) override;

private:
  void writeValue(const Value& value);

  String document_;
  bool yamlCompatibilityEnabled_{false};
  bool dropNullPlaceholders_{false};
  bool omitEndingLineFeed_{false};
};
```
**使用示例**
```c++
// 将数据序列化 -> 单行
// 进行数据的网络传输
std::string Json::FastWriter::write(const Value& root);
```
## Reader类
```c++
class JSON_API Reader {
public:
  using Char = char;
  using Location = const Char*;

  /** \brief An error tagged with where in the JSON text it was encountered.
   *
   * The offsets give the [start, limit) range of bytes within the text. Note
   * that this is bytes, not codepoints.
   */
  struct StructuredError {
    ptrdiff_t offset_start;
    ptrdiff_t offset_limit;
    String message;
  };

  /** \brief Constructs a Reader allowing all features for parsing.
    * \deprecated Use CharReader and CharReaderBuilder.
   */
  Reader();

  /** \brief Constructs a Reader allowing the specified feature set for parsing.
    * \deprecated Use CharReader and CharReaderBuilder.
   */
  Reader(const Features& features);

  /** \brief Read a Value from a <a HREF="http://www.json.org">JSON</a>
   * document.
   *
   * \param      document        UTF-8 encoded string containing the document
   *                             to read.
   * \param[out] root            Contains the root value of the document if it
   *                             was successfully parsed.
   * \param      collectComments \c true to collect comment and allow writing
   *                             them back during serialization, \c false to
   *                             discard comments.  This parameter is ignored
   *                             if Features::allowComments_ is \c false.
   * \return \c true if the document was successfully parsed, \c false if an
   * error occurred.
   */
  bool parse(const std::string& document, Value& root,
             bool collectComments = true);

  /** \brief Read a Value from a <a HREF="http://www.json.org">JSON</a>
   * document.
   *
   * \param      beginDoc        Pointer on the beginning of the UTF-8 encoded
   *                             string of the document to read.
   * \param      endDoc          Pointer on the end of the UTF-8 encoded string
   *                             of the document to read.  Must be >= beginDoc.
   * \param[out] root            Contains the root value of the document if it
   *                             was successfully parsed.
   * \param      collectComments \c true to collect comment and allow writing
   *                             them back during serialization, \c false to
   *                             discard comments.  This parameter is ignored
   *                             if Features::allowComments_ is \c false.
   * \return \c true if the document was successfully parsed, \c false if an
   * error occurred.
   */
  bool parse(const char* beginDoc, const char* endDoc, Value& root,
             bool collectComments = true);

  /// \brief Parse from input stream.
  /// \see Json::operator>>(std::istream&, Json::Value&).
  bool parse(IStream& is, Value& root, bool collectComments = true);

  /** \brief Returns a user friendly string that list errors in the parsed
   * document.
   *
   * \return Formatted error message with the list of errors with their
   * location in the parsed document. An empty string is returned if no error
   * occurred during parsing.
   * \deprecated Use getFormattedErrorMessages() instead (typo fix).
   */
  JSONCPP_DEPRECATED("Use getFormattedErrorMessages() instead.")
  String getFormatedErrorMessages() const;

  /** \brief Returns a user friendly string that list errors in the parsed
   * document.
   *
   * \return Formatted error message with the list of errors with their
   * location in the parsed document. An empty string is returned if no error
   * occurred during parsing.
   */
  String getFormattedErrorMessages() const;

  /** \brief Returns a vector of structured errors encountered while parsing.
   *
   * \return A (possibly empty) vector of StructuredError objects. Currently
   * only one error can be returned, but the caller should tolerate multiple
   * errors.  This can occur if the parser recovers from a non-fatal parse
   * error and then encounters additional errors.
   */
  std::vector<StructuredError> getStructuredErrors() const;

  /** \brief Add a semantic error message.
   *
   * \param value   JSON Value location associated with the error
   * \param message The error message.
   * \return \c true if the error was successfully added, \c false if the Value
   * offset exceeds the document size.
   */
  bool pushError(const Value& value, const String& message);

  /** \brief Add a semantic error message with extra context.
   *
   * \param value   JSON Value location associated with the error
   * \param message The error message.
   * \param extra   Additional JSON Value location to contextualize the error
   * \return \c true if the error was successfully added, \c false if either
   * Value offset exceeds the document size.
   */
  bool pushError(const Value& value, const String& message, const Value& extra);

  /** \brief Return whether there are any errors.
   *
   * \return \c true if there are no errors to report \c false if errors have
   * occurred.
   */
  bool good() const;

private:
  enum TokenType {
    tokenEndOfStream = 0,
    tokenObjectBegin,
    tokenObjectEnd,
    tokenArrayBegin,
    tokenArrayEnd,
    tokenString,
    tokenNumber,
    tokenTrue,
    tokenFalse,
    tokenNull,
    tokenArraySeparator,
    tokenMemberSeparator,
    tokenComment,
    tokenError
  };
  ```

  **使用示例**
```c++
bool Json::Reader::parse(const std::string& document,
    Value& root, bool collectComments = true);
    参数:
        - document: json格式字符串
        - root: 传出参数, 存储了json字符串中解析出的数据
        - collectComments: 是否保存json字符串中的注释信息

// 通过begindoc和enddoc指针定位一个json字符串
// 这个字符串可以是完成的json字符串, 也可以是部分json字符串
bool Json::Reader::parse(const char* beginDoc, const char* endDoc,
    Value& root, bool collectComments = true);
	
// write的文件流  -> ofstream
// read的文件流   -> ifstream
// 假设要解析的json数据在磁盘文件中
// is流对象指向一个磁盘文件, 读操作
bool Json::Reader::parse(std::istream& is, Value& root, bool collectComments = true);
```


