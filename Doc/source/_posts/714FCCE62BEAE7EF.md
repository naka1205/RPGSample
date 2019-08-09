---
title: 读入地图数据
date: 2019-05-27 13:15:45
tags: [战斗]
comment: false
categories: 战斗
createtime: 1558939214
description: 第1章到第5章的程序实例都是直接把人物数量和位置排列等写在程序里面,不过如果能改从配置文件读入这些相关信息的话会更方便。要是每次修改敌方人数或位置排列就要重新修改、创建程序,不仅事倍功半,而且万一游戏原始设计师跟写程序的设计师刚好是分工合作,不只有一个人可能修改这个源程序时,很容易乱成一团。最根本的问题是,游戏原始设计师也必须具备程序设计的基本常识和一套完整的开发操作系统。
---
###读入地图数据

第1章到第5章的程序实例都是直接把人物数量和位置排列等写在程序里面,不过如果能改从配置文件读入这些相关信息的话会更方便。

要是每次修改敌方人数或位置排列就要重新修改、创建程序,不仅事倍功半,而且万一游戏原始设计师跟写程序的设计师刚好是分工合作,不只有一个人可能修改这个源程序时,很容易乱成一团。最根本的问题是,游戏原始设计师也必须具备程序设计的基本常识和一套完整的开发操作系统。

####写配置文件

如果我们决定要使用专业的工具软件(如地图编辑器等),整个程序写起来会大费周折,所以暂时先在“文本文件”设置就好。到时候真的有必要的话,大不了再另外写一个能产生text格式配置文件的工具软件。请别误会,并不是说只是文本文件而已,就不该使用工具软件。

配置文件(map1.txt)

```
; map data

mapsize 10 10	;地图大小
mapimage map1	;地图CG

;伙伴
player 1 "主公" fighter
player 2 "魔法师" witch
player 3 "神官" priest

;敌人
enemy 1 "怪物" eye
enemy 2 "泥巴怪" puyo

;障碍物
object 1 "树林" tree2

;地图初始值
;
;■ 禁止出入的地区
;□ 可任意出入
;一	player 1
;二	player 2
;三	player 3
;四	player 4
;１	enemy 1
;２	enemy 2
;３	enemy 3
;４	enemy 4
;Ⅰ	object 1
;Ⅱ	object 2
;Ⅲ	object 3
;
map
□□□□□□□□Ⅰ□
□□□□ⅠⅠ□２□Ⅰ
□□□□□□□２□□
□二□□□□□□１□
□Ⅰ一□□Ⅰ□□１□
□三□□□□□□１□
□□□□□□□□１□
□□□□ⅠⅠ□２□□
□□□□□□□２□Ⅰ
□□□□□□□□Ⅰ□
```

利用这个配置文件所设定的项目有:
* 地图大小: mapsize 宽 高
* 地图背景CG: mapimage 文件名
* 游戏玩家的人物: player编号 姓名 CG文件名
* 敌对的人物 : enemy编号 姓名 CG文件名
* 障碍物: object编号 姓名 CG文件名
* 初始位置排列: map

地图初始位置排列中的字符和方格的对应关系如下:
■ : 禁止出入的地区
□ : 可任意出入
一~九 : 对应到player的“编号”
1~9 : 对应到enemy的“编号”
I~IX:对应到object的“编号”

因为现在还不用画面卷轴调整地图大小,所以是固定成“10×10”,但程序还是要写成能自由设定,如需修改时也比较方便。

除了初始位置排列之外,所有设定项目都是单行单项,只要逐行处理即可。对了,“:”后面的字符串是注解。

####读入文本文件

前面已经把配置文件做成文本文件的格式,所以现在需要有能读入并分析文本文件的类。

#####读入文本文件
**TextReader.h/TextReader.cpp**

先做一个读入文本文件的类(Tex(Reader类)。

读入文本文件(TextReader.h)

```C++
#ifndef	__TEXTREADER_H__
#define	__TEXTREADER_H__
//
// Script数据
//
class ScriptData {
  public:
	ScriptData(): status(0), lineno(0), string(0) {}
	ScriptData(const char *ptr, int no): status(0), lineno((unsigned short)no), string(ptr) {}

  public:
	unsigned char status;
	unsigned short lineno;
	const char *string;
} ;

//
// 读入script文件的类
//
class TextReader {
  public:
	TextReader(const char *name);
	~TextReader();

	// 取得已储存文件的文件名
	const char *GetFileName() const { return filename.c_str(); }
	// 读入1行
	const char *GetString() { return data.size() > read_index? data[read_index++].string: 0; }
	// 取得最后读入该行的行号
	int GetLineNo() const { return data[read_index-1].lineno; }
	// 文件是否已读取完毕
	bool IsOk() const { return buffer != 0; }
	operator bool() const { return IsOk(); }

	// 读入位置数据
	int GetPosition() const { return read_index; }
	// 设定位置数据
	void SetPosition(int position) { read_index = position; }
	// 返回1个读出位置
	void Back() { read_index--; }

  private:
	string filename;
	vector<ScriptData> data;
	char *buffer;
	int read_index;
} ;

#endif
```

读入文本文件(TextReader.cpp)

```C++
#include "StdAfx.h"
#include "TextReader.h"
//
// 构造函数
//
TextReader::TextReader(const char *name)
	:filename(name), read_index(0), buffer(0)
{
	CSecurityAttributes	sa;
	HANDLE hFile = ::CreateFile(name, GENERIC_READ, 0, &sa, OPEN_EXISTING,
		FILE_ATTRIBUTE_NORMAL | FILE_FLAG_SEQUENTIAL_SCAN, 0);
	if (hFile != INVALID_HANDLE_VALUE) {
                // 求出档案大小
		DWORD length = ::GetFileSize(hFile, 0);
		buffer = new char [length + 1];
		DWORD nread;
		if (!::ReadFile(hFile, buffer, length, &nread, 0)
                || length != nread) {	// error
			delete buffer;
			buffer = 0;
		}
		else {
			char *endptr = buffer + length;
			char *p = buffer;
			int lineno = 1;
			*endptr = '\0';		// 卫兵
			while (p < endptr && *p != '\027') {
				data.push_back(ScriptData(p, lineno));
				while (UC(*p) >= ' ' || *p == '\t')
					p++;

				if (*p == '\0')		// 最后没有换行符
					break;

				if (*p == '\r')		// Windows的换行符是"/r/n"
					*p++ = '\0';	// 所以字符串后面要以'/r'结尾
				if (*p == '\n')
					*p++ = '\0';
				lineno++;
			}
		}
		::CloseHandle(hFile);
	}
	data.push_back(ScriptData("end", 0));	// 最后面多加一个空的"end"
}
//
// 析构函数
//
TextReader::~TextReader()
{
	delete buffer;
}
```

以构造函数读入所有的文件内容,存储到vector。然后再用GetString成员函数逐行读出。

GetFileName和GetLineNo分别是出现错误消息时显示该文件名以及错误行的成员函数。文本文件既然是人动手写成的东西,几乎是百分之百会有几个错误,因此错误消息必须让看的人容易看出错误发生在哪里。

TextReader类的参照则如下所示。
```C++
TextReader::TextReader(constcharasut*name);
```
构造函数。把文本文件文件名设定到name。
```C++
const char *TextReader::GetFileName()const;
```
取得已有存储的文件名。
```C++
const char *TextReader:GetString();
```
读入1行的字符串,读入位置移到下一行。
```C++
int TextReader:GetLineNo() const;
```
取得最后读入该行的行号。
```C++
bool TextReader:IsOk() const;
```
若己读入文件则为true,若发生错误则返回false。
```C++
int TextReader:GetPosition() const:
```
返回当前的读入位置。
```C++
void TextReader:SetPosition(int position);
```
把读入位置设到指定位置(position)。
```C++
void TextReader::Back();
```
返回1个读入位置。

读入文本文件(TextReader)类和接下来要说明的词法分析(Lexer)类也会用在读入“情境脚本(scenarioscript,详见下一章的说明)”(甚至可以说这里只是借用人家专用的类而已)。因此,有些地图配置文件的读入会又臭又长。

#####词法分析(lexicalanalyze)

**Lexer.h/Lexer.cpp**

接着要做一个词法分析用的类。词法分析听起来好像工程浩大,其实也没那么夸张,只是利用空白(space)把字符串区隔成单字。

为了让后面的处理好做事,分析过的单字都配给一个“字符串”或“数值串”的标识。若为字符串则配给“IsString”标识,若为数值串则配给“IsNumber”标识。

请见下面的字符串分析实例。

字符串: abc 123 xyz
* 单字	标识
* "abc"	IsString
* "123"	IsNumber
* "xyz"	IsString

字符串: ”123”
* 单字	标识
* "123"	IsString
* 若数字前后有“””,则视为字符串

字符串:“123 xyz”
* 单字   标识
* "123 xyz"	IsString
* 若空白的前后有“叶”,则视为同一字符串

字符串:abc-xyz
* 单字   标识
* "abc" IsString
* "-" IsMinus
* "xyz" IsString
* 中间有符号,则视为不同字符串

从上面几个实例可知,这不只是用空白分割而已,处理时还要根据字符种类更改标识,或前后加上“11”让它里面也能有空白。

程序语言中所称的“词法分析”通常会需要进行保留字判断,或有更复杂的处理动作。它的处理比单纯的单字分割(tokendivision,记号分割)更复杂,这里所介绍的实例刚好属于比较中等的处理。

词法分析(Lexer.h)
```C++
#ifndef	__lexer_h__
#define	__lexer_h__
//
// 储存分析结果的结构体
//
struct LexValue	{
	string value;
	int type;
} ;
//
// Lexer类
//
class Lexer	{
  public:
	// 记号种类
	enum	{
		IsNumber,
		IsString,
		IsDelimitter,
		IsLabel,
		IsMinus,
	} ;

  private:
	enum	{
		IsSpace = IsMinus + 1,
		IsTerminater,
		IsQuotation,
	} ;

  public:
	Lexer(const char *str);

	int NumToken() const { return nToken; }
	void NextToken() { Count++; }

	const char *GetString(int index=-1);
	bool GetValue(int *value, int index=-1);
	int GetType(int index=-1);

  protected:
	const char *SkipSpace(const char *p);
	int CharType(unsigned char ch);
	bool MatchType(int &type, unsigned char ch);

  protected:
	int			nToken;
	vector<LexValue> Value;
	int			Count;
} ;

// inline成员函数
inline const char *Lexer::GetString(int index)
{
	if (index >= 0)
		Count = index;
	if (Count >= nToken)
		return NULL;
	return Value[Count++].value.c_str();
}

inline int	Lexer::GetType(int index)
{
	if (index >= 0)
		Count = index;
	if (Count >= nToken)
		return -1;
	return Value[Count].type;
}

#endif
```

词法分析(Lexer.cpp)

```C++
#include "StdAfx.h"
#include <mbctype.h>
#include "Lexer.h"

//
// 忽略空白字符
//
const char *Lexer::SkipSpace(const char *p)
{
	while (*p && isspace(*p))
		p++;
	return p;
}

//
// 判断字符种类
//
int Lexer::CharType(unsigned char ch)
{
	if (ch == '\0' || ch == '\n')
		return IsTerminater;
	if (isdigit(ch))
		return IsNumber;
	if (isalpha(ch) || _ismbblead(ch) || ch == '_')
		return IsString;
	if (isspace(ch))
		return IsSpace;
	if (ch == '"')
		return IsQuotation;
	if (ch == '-')
		return IsMinus;
	return IsDelimitter;
}

//
// 字符种类是否相同？
//
bool Lexer::MatchType(int &type, unsigned char ch)
{
	int	t = CharType(ch);
	switch (type) {
	  case IsLabel:
		if (ch == '*')
			return true;
		// no break

	  case IsNumber:
		if (t == IsString)
			type = IsString;
		// no break

	  case IsString:
		return (t == IsString || t == IsNumber);
	}
	return type == t;
}
//
// 记号切割
//
Lexer::Lexer(const char *str)
{
	for (nToken = 0; ; nToken++) {
		str = SkipSpace(str);
		if (*str == '\0' || *str == ';')
			break;
		int type = CharType(*str);

		if (type == IsTerminater && type == IsSpace)
			break;

		LexValue	value;

		if (type == IsQuotation) {
			value.type = IsString;
			str++;
			while (*str && CharType(*str) != IsQuotation) {
				if (_ismbblead(*str))
					value.value += *str++;
				value.value += *str++;
			}
			if (*str)
				str++;
		}
		else {
			if (*str == '-' && CharType(str[1]) == IsNumber) {
				value.value += '-';
				value.type = IsMinus;
				str++;
			}
			else {
				if (*str == '*' && nToken == 0)
					type = IsLabel;

				value.type = type;
				while (*str && MatchType(type, *str)) {
					if (_ismbblead(*str))
						value.value += *str++;
					value.value += *str++;
				}
				if (value.type == IsNumber)
					value.type = type;
			}
		}
		Value.push_back(value);
	}
	Count = 0;
}
//
// 读入数值
//
bool Lexer::GetValue(int *value, int index)
{
	bool	minus = false;
	int type = GetType(index);
	if (type == IsMinus) {
		minus = true;
		NextToken();
		type = GetType();
	}
	if (type != IsNumber)
		return false;
	const char *p = GetString();
	if (p == 0)
		return false;

	char *q;
	int v = strtol(p, &q, 10);
	*value = minus? -v: v;

	return true;
}
```

字符串分割的准则有“每次读出时只分割1个单字”或“先分割好再一个个读出”两种方法,这里采用后者,把所有处理工作都交给构造函数。

在这个实例中,可以先检查有多少个单字。把检查结果存储在nToken,就可以利用NumToken函数读出。

Lexer类的参数则如下所示。
```C++
Lexer::Lexer(const char *str);
```
构造函数。str是分析的字符串。
```C++
int Lexer::NumToken() const;
```
返回已分割的记号数。
```C++
void Lexer::NextToken();
```
记号的读出位置移到下一个记号。
```C++
const char Lexer:GetString(int index=-1);
```
读出设定位置的记号。若设定-1,则从目前位置读出。
```C++
boolLexer::GetValue(int*value,intindex=-1);
```
将设定位置的记号读出成数值。若设定-1,则从目前位置读出。若记号非数字类型,则返回false。
```C++
int Lexer:GetType(intindex=-1);
```
返回设定位置的记号种类。若设定-1,则从目前位置读出。以下则为保护(protect)成员函数的参数。
```C++
const char *Lexer::SkipSpace(const char *p);
```
忽略空白字符。
```C++
int Lexer:CharType(unsigned char ch);
```
取得该字符的字符种类。
```C++
bool Lexer::MatchType(int &type,unsigned char ch);
```
判断该字符是否为同一字符种类。

####成升类加入读入配置文件的功

能这样就能先把文本文件分割成单字(记号)后再读入。我们再应用它修改战斗类的初始化部分,就可以从配置文件读入地图数据。

战斗类(Battle.cpp)
```C++
//
// 读入战斗地图的资料
//
bool CBattleAction::Load(const char *name)
{
	// 若仍残留之前的状态，则为error
	// 应该不会有这个状态，故设为ASSERT
	ASSERT(map_data == 0);
	ASSERT(_map_data == 0);

	// 读入地图资料
	if (!LoadMap(name))
		return false;

	// 读入CG资料
	if (!Cursor.LoadImage("cursor")
	 || !Command.LoadImage("command")
	 || !TurnEnd.LoadImage("turnend")
	 || !StatusBase.LoadImage("status")
	 || !PopupImageBase.LoadImage("popup")
	 || !PopupParts.LoadImage("popup_parts")
	 || !FireBallImage.LoadImage("fireball")
	 || !HealImage.LoadImage("heal")
	 || !Explosion.LoadImage("explosion"))
		return false;
	// 产生图像
	CClientDC	dc(Parent);
	if (!PopupImage.Create(dc, PopupImageBase.Width(), PopupImageBase.Height()))
		return false;

	// 设定sprite
	CursorSprite.Set(&Cursor, CPoint(0, 0), CSize(MAPGRID_WIDTH, MAPGRID_HEIGHT), CMapSprite::DEPTH_CURSOR);

	{
		for (int i=0; i<MAX_COMMAND; i++) {
			CommandSprite[i].Set(&Command, CPoint(0, 0), CPoint(0, i * 22), CSize(70, 22), 0);
			CommandSprite[i].Show(false);
		}
	}
	TurnEndSprite.Set(&TurnEnd, CPoint(0, 0), CPoint(0, 0), CSize(70, 22), 0);
	PopupSprite.Set(&PopupImage, CPoint(0, 0), CPoint(0, 0), PopupImage.Size(), 0);

	// 人物的状态
	int total_width = StatusBase.Width() * MAX_CHARACTER + 3 * (MAX_CHARACTER - 1);
	int x = (WindowWidth - total_width) / 2;
	int y = WindowHeight - StatusBase.Height() - 10;
	{
		for (int i=0; i<MAX_CHARACTER; i++) {
			if (!Status[i].Create(dc, StatusBase.Width(), StatusBase.Height()))
				return false;
			StatusSprite[i].Set(Status + i, CPoint(x, y), CPoint(0, 0), StatusBase.Size(), 0);
			x += StatusBase.Width() + 3;
			SetPlayerStatus(i);
		}
	}

	CursorPos.x = -1;
	CursorPos.y = -1;
	SelPos.x = -1;
	SelPos.y = -1;
	SelChar = 0;
	status = STATUS_NONE;
	turn = PLAYER_TURN;
	NumberView = 0;

	StatusChar = 0;

	DrawMap(ViewImage);

	Parent->InvalidateRect(NULL);

	return true;
}
```

以上为初始化的函数。跟前面的初始化动作比起来,增加了攻击用的CG数据和sprite,不过基本上都是Init的扩充或修改后的产物。

读入真正地图配置文件的是下面这个函数。

读入配置文件的函数(Battle.cpp)

```C++
//
// 读入地图定义档
//
bool CBattleAction::LoadMap(const char *name)
{
	typedef bool (CBattleAction::*cmd_t)(TextReader &, Lexer &);
	typedef pair<const char *, cmd_t>	CmdTab;
	typedef map<const char *, cmd_t, ic_less>	cmdmap;
	cmdmap	cmd_table;

	cmd_table.insert(CmdTab("player",	&CBattleAction::PlayerCmd));
	cmd_table.insert(CmdTab("enemy",	&CBattleAction::EnemyCmd));
	cmd_table.insert(CmdTab("object",	&CBattleAction::ObjectCmd));
	cmd_table.insert(CmdTab("mapsize",	&CBattleAction::MapSizeCmd));
	cmd_table.insert(CmdTab("mapimage",	&CBattleAction::MapImageCmd));
	cmd_table.insert(CmdTab("map",		&CBattleAction::MapCmd));

	char	path[_MAX_PATH];
	sprintf(path, MAPPATH "%s.txt", name);

	TextReader	reader(path);
	if (!reader) {
		Parent->MessageBox("无法读取地图档案。");
		return false;
	}

	const char *str;
	while ((str = reader.GetString()) != 0) {
		Lexer	lexer(str);

		if (lexer.NumToken()) {
			const char *cmd = lexer.GetString(0);
			if (stricmp(cmd, "end") == 0)	// 结束
				break;

			cmdmap::iterator p = cmd_table.find(cmd);
			if (p != cmd_table.end()) {
				if (!(this->*p->second)(reader, lexer))
					return false;
			}
			else {
				Error(reader, "语法错误");
				return false;
			}
		}
	}
	return true;
}
```

以下依次说明读入配置文件的函数。首先,为了简化程序的叙述内容,先用typedef定义“类型”。
```C++
typedef bool(CBattleAction:*cmd_t(TextReader &,lexer &);
```
这个是指到成员函数的函数指针。函数指针不仅叙述过长,而且又难懂,所以另外再给一个别名。

说明一下,别名是用“cmd_t”的“bool(CBattleAction:*)(TextReaderLexer &)类型”,也就是把“(TextReader &,Lexer&)”)放到引用变量,然后返回bool的CbattleAction的成员函数指针类型。
```C++
typedef pair<const char *,cmd_t>CmdTab:
```
命令表格(commandtable)的元素之一。第1个“constchar*”是命令名称,第二个“cmd_t”则是对应的“命令处理函数”。
```C++
typedef map<const char *,cmd_t,ic_less> cmdmap;
```
STL的map内容的别名。第1个、第2个和CmdTab是同一元素。第3个则是进行比较的函数对象。

函数对象则定义在less.h里。

比较字符串(less.h)
```C++
#ifndef	__less_h__
#define	__less_h__

class ic_less: public binary_function<const char *, const char *, bool> {
  public:
	result_type operator()(first_argument_type x, second_argument_type y) const
	{
		return stricmp(x, y) < 0;
	}
} ;

class ics_less: public binary_function<const string &, const string &, bool> {
  public:
	result_type operator()(first_argument_type x, second_argument_type y) const
	{
		return stricmp(x.c_str(), y.c_str()) < 0;
	}
} ;
#endif
```

比较设定为不分大小写,两者一视同仁。主要是因为Windows不会去区分命令、文件名的大小写(早年的MS-DOS也是如此),或者是很多人根本就懒得去分辨(这是笔者自己的经验谈,如果你坚持要分大小写也没什么不妥)。

如果比较字符串要区分大小写,(一般来说)多半使用“strcmp”,但这里使用的是“stricmp”。其引用变量和返回值的意义均同strcmp。

```C++
	cmdmap	cmd_table;

	cmd_table.insert(CmdTab("player",	&CBattleAction::PlayerCmd));
	cmd_table.insert(CmdTab("enemy",	&CBattleAction::EnemyCmd));
	cmd_table.insert(CmdTab("object",	&CBattleAction::ObjectCmd));
	cmd_table.insert(CmdTab("mapsize",	&CBattleAction::MapSizeCmd));
	cmd_table.insert(CmdTab("mapimage",	&CBattleAction::MapImageCmd));
	cmd_table.insert(CmdTab("map",		&CBattleAction::MapCmd));
```

把“所有命令”登录到cmd_table。命令就是前面说明的配置文件命令,顺便也把处理该命令的函数登录进去。
命令处理的分派

```C++
const char *str;
while ((str = reader.GetString()) != 0) {
	Lexer	lexer(str);

	if (lexer.NumToken()) {
		const char *cmd = lexer.GetString(0);
		if (stricmp(cmd, "end") == 0)	// 结束
			break;

		cmdmap::iterator p = cmd_table.find(cmd);
		if (p != cmd_table.end()) {
			if (!(this->*p->second)(reader, lexer))
				return false;
		}
		else {
			Error(reader, "语法错误");
			return false;
		}
	}
}
```

读出1行后,就在Lexer类别分解成单字。若单字数目为“0”即为空行(或该行只有注解),故不做任何动作。

若第1个单字为“end”即结束,否则到cmdtable进行搜寻,找到后再调用有登录在表格内的函数。
```C++
cmdmap::iterator p = cmd_table.Find(cmd);
```
如果找不到map::find,则返回end(),故若非end()则表示有找到。
```C++
if(!(this->p->secoud)(reader,lexar))
	return false;
```
调用成员函数。p->second会返回第2个元素(函数指针)。

调用成员函数的指针时的调用方法是“(this->*函数指针)(引数)”,所以加起来就是“(this->*p->second)(引数)”。

由于这里的命令种类不多,也许有人会认为应该可以写成:

```C
bool result;
if(stricmp("player",cmd) == 0){
	result = PlayerCmd(reader,lexar);
}
else if(stricmp("enemy",cmd) == 0){
	result = EnemyCmd(reader,lexar);
}
```

也不是说不可以,但是考虑到将来想要新增或修改命令的话,这种“利用表格的方法”就有很大的优点,到时候只要直接增减表格就能同时修改命令的处理部分。

所有命令的处理都要做成函数。因为这个结构有使用到表格,故命令处理函数的引数和返回值必须设为同一类型。

游戏玩家CG的登录命令(Battle.cpp)

```C
bool CBattleAction::PlayerCmd(TextReader &reader, Lexer &lexer)
{
	int number;
	bool b = lexer.GetValue(&number);
	const char *name = lexer.GetString();
	const char *file = lexer.GetString();

	if (name == 0 || file == 0 || !b || lexer.GetString() != 0) {
		Error(reader, "语法错误");
		return false;
	}
	CChip chip(name, file);
	if (chip.image == 0) {
		Error(reader, "无法读取CG数据");
		return false;
	}
	chip_list.insert(pair<int, CChip>(PlayerNumber(number - 1), chip));

	return true;
}
```

这是登录游戏玩家CG的命令。读出CG后,先登录到chip_list。

敌人CG的登录命令(Battle.cpp)

```C++
bool CBattleAction::EnemyCmd(TextReader &reader, Lexer &lexer)
{
	int number;
	bool b = lexer.GetValue(&number);
	const char *name = lexer.GetString();
	const char *file = lexer.GetString();

	if (name == 0 || file == 0 || !b || lexer.GetString() != 0) {
		Error(reader, "语法错误");
		return false;
	}
	CChip chip(name, file);
	if (chip.image == 0) {
		Error(reader, "无法读取CG数据");
		return false;
	}
	chip_list.insert(pair<int, CChip>(EnemyNumber(number - 1), chip));

	return true;
}
```

障碍物CG的登录命令(Battle.cpp)

```C++
bool CBattleAction::ObjectCmd(TextReader &reader, Lexer &lexer)
{
	int number;
	bool b = lexer.GetValue(&number);
	const char *name = lexer.GetString();
	const char *file = lexer.GetString();

	if (name == 0 || file == 0 || !b || lexer.GetString() != 0) {
		Error(reader, "语法错误");
		return false;
	}
	CChip chip(name, file);
	if (chip.image == 0) {
		Error(reader, "无法读取CG数据");
		return false;
	}
	chip_list.insert(pair<int, CChip>(ObjectNumber(number - 1), chip));

	return true;
}
```
这是登录敌人和障碍物CG的命令。这里的处理跟登录游戏玩家CG时一样,只是登录在chip_list的号码不一样而已。

设定地图大小的命令(Battle.cpp)

```C++
bool CBattleAction::MapSizeCmd(TextReader &reader, Lexer &lexer)
{
	int width, height;
	if (!lexer.GetValue(&width)
	 || !lexer.GetValue(&height)
	 || lexer.GetString() != 0) {
		Error(reader, "语法错误");
		return false;
	}
	MapSize.cx = width;
	MapSize.cy = height;

	MapRect.SetRect(0, 0, width, height);

	// 保留map_data的空间
	_map_data = new MapData[width * height];
	map_data = new MapData *[height];

	for (int i=0; i<height; i++) {
		map_data[i] = _map_data + i * width;
	}

	return true;
}

```

上为设定地图尺寸大小的命令。一开始要先保留地图的可用空间。保留内存空间的方法则如前章Init所示,利用(类似)二维数组的方式保留可用空间。

读入地图背景CG(Battle.cpp)

```C++
bool CBattleAction::MapImageCmd(TextReader &reader, Lexer &lexer)
{
	const char *name = lexer.GetString();

	if (name == 0 || lexer.GetString() != 0) {
		Error(reader, "语法错误");
		return false;
	}
	if (!MapImage.LoadImage(name)) {
		Error(reader, "无法读取CG数据");
		return false;
	}
	return true;
}
```

以上程序为读入地图背景的CG。

地图的初始位置排列(Battle.cpp)
```C++
//
// 把map上的文字改成状态
//
int CBattleAction::FindMapStatus(const char *find)
{
	struct {
		char str[4];
		unsigned type;
	} table[] = {
		{ "□", NONE },
		{ "■", RESTRICTED_AREA },
		{ "一", PLAYER | (0 << 8) },
		{ "二", PLAYER | (1 << 8) },
		{ "三", PLAYER | (2 << 8) },
		{ "四", PLAYER | (3 << 8) },
		{ "五", PLAYER | (4 << 8) },
		{ "六", PLAYER | (5 << 8) },
		{ "七", PLAYER | (6 << 8) },
		{ "八", PLAYER | (7 << 8) },
		{ "九", PLAYER | (8 << 8) },
		{ "１", ENEMY | (0 << 8) },
		{ "２", ENEMY | (1 << 8) },
		{ "３", ENEMY | (2 << 8) },
		{ "４", ENEMY | (3 << 8) },
		{ "５", ENEMY | (4 << 8) },
		{ "６", ENEMY | (5 << 8) },
		{ "７", ENEMY | (6 << 8) },
		{ "８", ENEMY | (7 << 8) },
		{ "９", ENEMY | (8 << 8) },
		{ "Ⅰ", OBJECT | (0 << 8) },
		{ "Ⅱ", OBJECT | (1 << 8) },
		{ "Ⅲ", OBJECT | (2 << 8) },
		{ "Ⅳ", OBJECT | (3 << 8) },
		{ "Ⅴ", OBJECT | (4 << 8) },
		{ "Ⅵ", OBJECT | (5 << 8) },
		{ "Ⅶ", OBJECT | (6 << 8) },
		{ "Ⅷ", OBJECT | (7 << 8) },
		{ "Ⅸ", OBJECT | (8 << 8) },
	} ;
	for (int i=0; i<(sizeof(table) / sizeof(table[0])); i++) {
		if (strcmp(find, table[i].str) == 0)
			return table[i].type;
	}
	return -1;
}

bool CBattleAction::MapCmd(TextReader &reader, Lexer &lexer)
{
	if (map_data == 0) {	// 此时，已经要决定地图的大小
		Error(reader, "请先设定mapsize");
		return false;
	}

	for (int y=0; y<MapSize.cy; y++) {
		const char *str = reader.GetString();
		if (str == 0) {
			Error(reader, "语法错误");
			return false;
		}
		for (int x=0; x<MapSize.cx; x++) {
			char chip[3];
			chip[0] = str[x * 2 + 0];
			chip[1] = str[x * 2 + 1];
			chip[2] = 0;

			int idx = FindMapStatus(chip);
			if (idx < 0) {
				Error(reader, "语法错误");
				return false;
			}
			map_data[y][x].type = (unsigned char)idx;
			if (idx != RESTRICTED_AREA && idx != NONE) {
				map<int, CChip>::iterator p = chip_list.find(idx);
				if (p == chip_list.end()) {		// 尚未登录
					Error(reader, "人物角色尚未登录");
					return false;
				}
				else {
					TRACE("[%s] = %d, %d\n", p->second.name.c_str(), x, y);
					if ((idx & PLAYER) != 0) {
						player_list.push_back(CCharacter(p->second.image, x, y, CCharacter::DEPTH_CHAR));
						CCharacter &s = player_list.back();

						// 根据人物姓名读入状态
						CStatus *status = Parent->GetParam().GetStatus(p->second.name.c_str());
						if (status == 0) {	// 无的状态
							Error(reader, "找不到人物%s的状态信息");
							return false;
						}
						s.status = *status;

						sprite.insert(&s);
						s.SetDirection(CCharacter::RIGHT);
						map_data[y][x].sprite = &s;

						StatusCharacter[idx >> 8] = &s;	// 显示用人物相关信息
					}
					else if ((idx & ENEMY) != 0) {
						enemy_list.push_back(CCharacter(p->second.image, x, y, CCharacter::DEPTH_CHAR));
						CCharacter &s = enemy_list.back();

						// 根据人物姓名读入状态
						CStatus *status = Parent->GetParam().GetStatus(p->second.name.c_str());
						if (status == 0) {	// 无的状态
							Error(reader, "找不到人物%s的状态信息");
							return false;
						}
						s.status = *status;

						sprite.insert(&s);
						s.SetDirection(CCharacter::LEFT);
						map_data[y][x].sprite = &s;
					}
					else if ((idx & OBJECT) != 0) {
						object_list.push_back(CMapSprite(p->second.image, x, y, CMapSprite::DEPTH_OBJECT));
						CMapSprite &s = object_list.back();
						sprite.insert(&s);
						map_data[y][x].sprite = &s;
					}
				}
			}
		}
	}
	return true;
}
```

把人物等放到地图上。配置文件的地图数据是“1个字符=1方格”,所以是一次处理1个字符。

```C++
char chip[3];
chip[0] = str[x * 2 + 0];
chip[1] = str[x * 2 + 1];
chip[2] = 0;

int idx = FindMapStatus(chip);
if (idx < 0) {
	Error(reader, "语法错误");
	return false;
}
```
1个中文字符也是双字节组(2bytes),所以先读出2bytes,把它从“字符”转换成“方格信息”(利用FindMapStatus)。

```C++
map_data[y][x].type = (unsigned char)idx;
```

这个数据的最后1byte会直接写入到地图里。

当地图上有人物或障碍物时(也就是非“什么都没有(NONE)”或“禁止非障碍物出入(RESTRICTED_AREA)”时),则登录到人物清单。

```C++
map<int, CChip>::iterator p = chip_list.find(idx);
if (p == chip_list.end()) {		// 尚未登录
	Error(reader, "人物角色尚未登录");
	return false;
}
```

chiplist是己用player或enemy命令完成登录的CG清单(小人物)。如果没有登录在里面,就表示player或enemy等的命令叙述有误。

```C++
if ((idx & PLAYER) != 0) {
	player_list.push_back(CCharacter(p->second.image, x, y, CCharacter::DEPTH_CHAR));
```

如果在方格上的是游戏玩家,则登录到play_list。

```C++
CCharacter &s = player_list.back();

// 根据人物姓名读入状态
CStatus *status = Parent->GetParam().GetStatus(p->second.name.c_str());
if (status == 0) {	// 无的状态
	Error(reader, "找不到人物%s的状态信息");
	return false;
}
s.status = *status;
```

读入人物的状态。状态是指人物的移动步数、HP等相关数据资料。这些状态会存储在CMainWin类里,即使战斗完毕,这个类别仍然会保留下来。

```C++
sprite.insert(&s);
s.SetDirection(CCharacter::RIGHT);
map_data[y][x].sprite = &s;
```

登录sprite。

```C++
StatusCharacter[idx >> 8] = &s;	// 显示用人物相关信息
```

如果是游戏玩家所选的人物,因为状态要显示在画面上,所以必须登录到状态显示用的变量里。敌人和障碍物的处理亦同。

当发生程序错误时,应有个会显示错误消息(含文件名和行号)的消息对话框函数,程序要调用这个函数。

误处理(Battle.cpp)
```C++
//
// 输出错误事件
//
void __cdecl CBattleAction::Error(TextReader &reader, const char *fmt, ...)
{
	va_list	args;

	char	str[256];
	int len = sprintf(str, "%s:%d ", reader.GetFileName(), reader.GetLineNo());
	va_start(args, fmt);
	_vsnprintf(str + len, sizeof(str) - len, fmt, args);
	va_end(args);

	Parent->MessageBox(str);
}
```

"_vsnprintf"是"vsnprintf"另一个“有字符数限制的版本”。字符数超过上限时会破坏堆栈(stack),造成程序操作错误,所以用_vsnprintf会比较保险一点。

只不过就算消息被半路腰斩,字符数超过上限的bug还是不变的事实。换个角度想,错误消息本来就是游戏开发人员才会看到的东西,所以不算是什么大问题(通常游戏玩家不会有机会看到这种错误消息)。

把前面这些处理整合起来,就可以从配置文件读入地图数据,显示战斗地图的初始状态。