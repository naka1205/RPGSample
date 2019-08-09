---
title: 参数编辑器
date: 2019-05-30 12:50:10
tags: [参数]
comment: false
categories: 编辑参数
createtime: 1559193317
description: 参数编辑器是仅供游戏程序设计师使用的“内部工具”。设计内部工具时不必考虑“好不好看”的问题,而且也不必为了DLL发布而另外写安装程序,所以写起来不会得手碍脚。
---
###参数编辑器

参数编辑器是仅供游戏程序设计师使用的“内部工具”。设计内部工具时不必考虑“好不好看”的问题,而且也不必为了DLL发布而另外写安装程序,所以写起来不会得手碍脚。

反向思考回来,如果写这个内部工具都已经焦头烂额的话,以后要修改规格时更不知该如何下手,因此千万不要写得太复杂。

####雏形

这里是套用VisualC++的向导功能,尽量降低工作量的负担。参数编辑器比较接近“商业应用软件”或“公用软件”,所以多利用MFC会比较好做。操作的程序步骤如下所示。

①先到VisualC++的菜单,选【File】-【New...】命令,如图8-1所示。

②跳出“New”的对话框后,选择“MFCAppWizard(exe)”并输入“ProjectName”,如图8-2所示。

③进入“MFCAppWizard-step1”,选取“Dialogbased”,如图8-3所示。

④在“MFC AppWizard-step2/4”输入“DialogTitle”,本例使用“参数编辑器”当标题,如图8-4所示。

⑤接着只要根据默认值进行下去,最后在上图单击“Finlsh”钮即可自动由向导产生预设的程序代码,如图8-5所示。

这样就是一个对话框(Dialog-Based)的雏形。这个实例我们取的ProjectName是“ParamEdit”,而在(4)的“DialogTitle”取名为“参数编辑器”。反正标题可以随时更换,如果还想不到什么好名字,先直接写“参数编辑器”也没关系。

对话框是指由“对话框”组成的应用软件,因为这里的交互式编辑数值,所以只要有对话框就能用。

####对话框

下一个操作是利用Visua1C++的"DialogEditor”制作对话框。参数名称的“标签”和编辑用的防解力“编辑器”、“下拉式列表框”要逐一列入,控制项的排列方式可参见图8-6。

所有元件都就定位之后,再对各个元件“按下鼠标右键”,选取【Propertis...】指令并修改ID。沿用默认值的ID比较不容易表达真正意思,例如“IDDEDIT1”等等,故请配合功能命名。

继续用“鼠标右键”菜单中的【ClassWizard(W)】令,启动“ClassWizard”,如图8-7所示。

每个控制项都要分配1个成员变量。附赠光盘的“chapter8”文件夹下有一个参数编辑器实例的project文件,有需要的人可参考使用。关于类ID的命名、成员变量分配方式等,则请参见该project文件。

到此,编辑画面已经差不多都完成了。虽然没有任何编写程序代码的工作,不过这些已经有固定格式的部分都可以利用向导自动帮我们来做。

####编写程序代码

利用向导能完成的部分只到画面而已,处理的内容还是需要自行编写代码。

#####读取/写入参数文件

第一个需要有的是读取/写入文件的处理,那就从这里开始吧!

读取/写入参数文件(ParamFile.h)

```C++
#ifndef	__PARAMFILE_H__
#define	__PARAMFILE_H__

#include <string>
#include <map>

class CParamFile {
  public:
	struct parameter {
		char	name[16];			// 人物名称
		short	level;				// 等级
		short	experience;			// 经验值
		short	move_dist;			// 移动距离
		short	attack_dist;		// 攻击距离
		short	attack_power;		// 攻击力
		short	magic_power;		// 魔法力
		short	defence_power;		// 防御力
		short	resistance;			// 抵抗力
		short	hit_point;			// HP
		short	magic_point;		// MP
		short	magic;				// 使用魔法
		short	filler;				// 补满
	} ;

  public:
	bool Load(const char *name);
	bool Save(const char *name);

	const parameter *Find(const char *name);
	void Add(const char *name, const parameter &param);

	void FindFirst();
	bool FindNext(char *name);

  protected:
	typedef std::map<std::string, parameter> params_t;
	params_t params;
	params_t::iterator iter;
} ;

#endif
```

读取/写入参数文件(ParamFile.cpp)

```C++
#include "StdAfx.h"
#include "ParamFile.h"
using namespace std;
//
// 从文件读取参数
//
bool CParamFile::Load(const char *name)
{
	FILE *fp = fopen(name, "rb");
	if (fp == 0)
		return false;

	long count;
	if (fread(&count, 1, sizeof(count), fp) == sizeof(count)) {
		for (long i=0; i<count; i++) {
			parameter p;
			if (fread(&p, 1, sizeof(p), fp) == sizeof(p)) {
				params[p.name] = p;
			}
		}
	}
	fclose(fp);

	return true;
}
//
// 把参数写进文件
//
bool CParamFile::Save(const char *name)
{
	FILE *fp = fopen(name, "wb");
	if (fp == 0)
		return false;

	long count = params.size();
	fwrite(&count, 1, sizeof(count), fp);
	for (params_t::iterator p = params.begin(); p != params.end(); ++p) {
		fwrite(&p->second, 1, sizeof(p->second), fp);
	}
	fclose(fp);

	return true;
}
//
// 搜寻内部表格中设定名称所对应的参数
//
const CParamFile::parameter *CParamFile::Find(const char *name)
{
	params_t::iterator p = params.find(name);
	if (p == params.end())	// 无此参数
		return 0;
	return &p->second;
}
//
// 登录到参数表格
//
void CParamFile::Add(const char *name, const CParamFile::parameter &param)
{
	params[name] = param;
}
//
// 准备名称一览表
//
void CParamFile::FindFirst()
{
	iter = params.end();
}
//
// 读出1个名称
//
bool CParamFile::FindNext(char *name)
{
	if (iter == params.end())
		iter = params.begin();
	else
		++iter;
	if (iter == params.end())
		return false;

	strncpy(name, iter->second.name, 16);
	return true;
}
```

稍微解释一下程序代码的内容。

**从文件读出参数-CParamFile::Load**

**将参数写入文件-CParamFile::Save**

参数文件须为“二进制文件”。

装入和存储处理是密不可分的组合,如欲修改时,当然两边都要同步修改。否则会无法读取既有文件。

**搜寻名称所对应的参数-CParamFile::Find**

参数表格是使用`std:map`。

以字符串为关键字进行搜寻时,男庸质疑应该使用map容器。万一情况不妙还可最后再改,而且STL的成员函数是“同一处理则用同一名称”,所以修改比较容易。在此若利用map容器做搜寻,亦可使用前面出现频率较高的find。

**登录到参数表格-CParamFile::Add**

利用下面这1行执行登录到参数表格的动作。

```C++
params [name] = param:
```

**取得名称一览表-CParamFile::FindFirst/CParamFile::FindNext**

名称一览表须登录到下拉式列表框,因此要有一个这样的功能。
这个函数是为了让类间的关系“不紧密”。

#####参数的编辑处理

接下来是编辑参数的相关处理。

编辑参数(ParamEditDlg.cpp)

```C++
// ParamEditDlg.cpp : 实现程序(implementation file)
//

#include "stdafx.h"
#include "ParamEdit.h"
#include "ParamEditDlg.h"
#include "AddDlg.h"

#ifdef _DEBUG
#define new DEBUG_NEW
#undef THIS_FILE
static char THIS_FILE[] = __FILE__;
#endif

/////////////////////////////////////////////////////////////////////////////
// CParamEditDlg 对话框(dialog)
CParamEditDlg::CParamEditDlg(CWnd* pParent /*=NULL*/)
	: CDialog(CParamEditDlg::IDD, pParent)
{
	//{{AFX_DATA_INIT(CParamEditDlg)
	m_attack = 0;
	m_attack_dist = 0;
	m_defence = 0;
	m_hp = 0;
	m_level = 0;
	m_magic_power = 0;
	m_move_dist = 0;
	m_mp = 0;
	m_resistance = 0;
	m_magic = -1;
	//}}AFX_DATA_INIT
	// 备注：LoadIcon不要求Win32的DestroyIcon的子序列
	// Note that LoadIcon does not require a subsequent DestroyIcon in Win32
	m_hIcon = AfxGetApp()->LoadIcon(IDR_MAINFRAME);
}

void CParamEditDlg::DoDataExchange(CDataExchange* pDX)
{
	CDialog::DoDataExchange(pDX);
	//{{AFX_DATA_MAP(CParamEditDlg)
	DDX_Control(pDX, IDC_NAME, m_name);
	DDX_Text(pDX, IDC_ATTACK, m_attack);
	DDV_MinMaxInt(pDX, m_attack, 1, 9999);
	DDX_Text(pDX, IDC_ATTACKDIST, m_attack_dist);
	DDV_MinMaxInt(pDX, m_attack_dist, 1, 10);
	DDX_Text(pDX, IDC_DEFENCE, m_defence);
	DDV_MinMaxInt(pDX, m_defence, 1, 9999);
	DDX_Text(pDX, IDC_HP, m_hp);
	DDV_MinMaxInt(pDX, m_hp, 1, 9999);
	DDX_Text(pDX, IDC_LEVEL, m_level);
	DDV_MinMaxInt(pDX, m_level, 1, 99);
	DDX_Text(pDX, IDC_MAGICPOWER, m_magic_power);
	DDV_MinMaxInt(pDX, m_magic_power, 1, 9999);
	DDX_Text(pDX, IDC_MOVEDIST, m_move_dist);
	DDV_MinMaxInt(pDX, m_move_dist, 1, 10);
	DDX_Text(pDX, IDC_MP, m_mp);
	DDV_MinMaxInt(pDX, m_mp, 1, 9999);
	DDX_Text(pDX, IDC_RESISTANCE, m_resistance);
	DDV_MinMaxInt(pDX, m_resistance, 1, 9999);
	DDX_CBIndex(pDX, IDC_MAGIC, m_magic);
	//}}AFX_DATA_MAP
}

BEGIN_MESSAGE_MAP(CParamEditDlg, CDialog)
	//{{AFX_MSG_MAP(CParamEditDlg)
	ON_WM_PAINT()
	ON_WM_QUERYDRAGICON()
	ON_CBN_SELCHANGE(IDC_NAME, OnSelchangeName)
	ON_BN_CLICKED(IDC_ADD, OnAdd)
	ON_BN_CLICKED(IDC_SET, OnSet)
	//}}AFX_MSG_MAP
END_MESSAGE_MAP()

#define PARAMFILE	"parameter.dat"

/////////////////////////////////////////////////////////////////////////////
// CParamEditDlg message handlers 事件处理程序

BOOL CParamEditDlg::OnInitDialog()
{
	CDialog::OnInitDialog();

	// 设定此对话框用的图形按钮。若应用软件的主窗口不是
	// 对话框时，则自动不设定框架
	// Set the icon for this dialog.  The framework does this automatically
	//  when the application's main window is not a dialog
	SetIcon(m_hIcon, TRUE);			// Set big icon
	SetIcon(m_hIcon, FALSE);		// Set small icon

	// TODO: 如欲进行特别的初始化时，请新增在这里
	// TODO: Add extra initialization here
	params.Load(PARAMFILE);

	char name[16];
	params.FindFirst();
	while (params.FindNext(name)) {
		m_name.AddString(name);
	}
	m_name.SetCurSel(0);
	ChangeName();

	return TRUE;  // 若传回TRUE，则保留控制项里所设定的反白光棒（focus）
	              // return TRUE  unless you set the focus to a control
}

// 如欲在对话盒内新增最小化按钮，则须有一段绘制图形按钮
// 的程序代码（如下所示）。由于MFC应用软件是使用document/view
// 模型，因此框架会自动去处理这个处理动作。
// If you add a minimize button to your dialog, you will need the code below
//  to draw the icon.  For MFC applications using the document/view model,
//  this is automatically done for you by the framework.

void CParamEditDlg::OnPaint() 
{
	if (IsIconic())
	{
						   // 绘制用的装置环境组态（DC：device context）
		CPaintDC dc(this); // device context for painting

		SendMessage(WM_ICONERASEBKGND, (WPARAM) dc.GetSafeHdc(), 0);

		// 用户端方形区域的正中央
		// Center icon in client rectangle
		int cxIcon = GetSystemMetrics(SM_CXICON);
		int cyIcon = GetSystemMetrics(SM_CYICON);
		CRect rect;
		GetClientRect(&rect);
		int x = (rect.Width() - cxIcon + 1) / 2;
		int y = (rect.Height() - cyIcon + 1) / 2;

		// 绘制图示
		// Draw the icon
		dc.DrawIcon(x, y, m_hIcon);
	}
	else
	{
		CDialog::OnPaint();
	}
}

// 为了在user拖曳缩到最小的窗口时能显示光标，
// 系统会调用这里
// The system calls this to obtain the cursor to display while the user drags
//  the minimized window.
HCURSOR CParamEditDlg::OnQueryDragIcon()
{
	return (HCURSOR) m_hIcon;
}

void CParamEditDlg::OnOK() 
{
	// TODO:其他验证用的程序代码要新增在此
	// TODO: Add your control notification handler code here
	params.Save(PARAMFILE);

	CDialog::OnOK();
}

void CParamEditDlg::OnSelchangeName() 
{
	// TODO:控制项通知处理程序用的程序代码要新增在此
	// TODO: Add your control notification handler code here	
	ChangeName();
}

void CParamEditDlg::OnAdd() 
{
	// TODO:控制项通知处理程序用的程序代码要新增在此
	// TODO: Add your control notification handler code here
	CAddDlg dlg;
	if (dlg.DoModal() == IDOK) {
		if (m_name.FindStringExact(-1, dlg.m_name) != CB_ERR) {
			MessageBox("这个名称与已经登录的名称重复！");
		}
		else {
			m_name.SetCurSel(m_name.AddString(dlg.m_name));
			ChangeName();
		}
	}
}

void CParamEditDlg::ChangeName()
{
	int index = m_name.GetCurSel();
	CString str;
	m_name.GetLBText(index, str);
	const CParamFile::parameter *p = params.Find(str);
	if (p == 0) {
		m_attack = 1;
		m_attack_dist = 1;
		m_defence = 1;
		m_hp = 1;
		m_level = 1;
		m_magic_power = 1;
		m_move_dist = 1;
		m_mp = 1;
		m_resistance = 1;
		m_magic = 0;
	}
	else {
		m_attack = p->attack_power;
		m_attack_dist = p->attack_dist;
		m_defence = p->defence_power;
		m_hp = p->hit_point;
		m_level = p->level;
		m_magic_power = p->magic_power;
		m_move_dist = p->move_dist;
		m_mp = p->magic_point;
		m_resistance = p->resistance;
		m_magic = p->magic;
	}
	UpdateData(FALSE);
}

void CParamEditDlg::OnSet() 
{
	// TODO:控制项通知处理程序用的程序代码要新增在此
	// TODO: Add your control notification handler code here

	if (UpdateData(TRUE)) {
		CParamFile::parameter p;

		int index = m_name.GetCurSel();
		CString str;
		m_name.GetLBText(index, str);

		memset(&p, sizeof(p), 0);
		strncpy(p.name, str, 16);
		p.attack_power = m_attack;
		p.attack_dist = m_attack_dist;
		p.defence_power = m_defence;
		p.hit_point = m_hp;
		p.level = m_level;
		p.magic_power = m_magic_power;
		p.move_dist = m_move_dist;
		p.magic_point = m_mp;
		p.resistance = m_resistance;
		p.magic = m_magic;

		params.Add(str, p);
	}
}
```

向导能把对话框的代码写到某个程度,所以再“新增”一些必要功能进去就好。

**读取文件**

文件的读取动作是在初始化对话框时(OnlnitDialog)进行的。

```C++
BOOL CParamEditDlg::OnInitDialog()
{
	...

	// TODO: 如欲进行特别的初始化时，请新增在这里
	// TODO: Add extra initialization here
	params.Load(PARAMFILE);

	char name[16];
	params.FindFirst();
	while (params.FindNext(name)) {
		m_name.AddString(name);
	}
	m_name.SetCurSel(0);
	ChangeName();

	return TRUE;  // 若传回TRUE，则保留控制项里所设定的反白光棒（focus）
	              // return TRUE  unless you set the focus to a control
}
```

使用AddString将名称登录到下拉式列表框,直到FindNext返回false为止。

登录结束后,选择第1个(SetCurSel),利用第1个所对应的参数初始化各个控制项(ChangeName)。

**写入文件**

写入操作是指当按下对话框的OK按钮时(OnOK)能够写入。

```C++
void CParamEditDlg::OnOK() 
{
	// TODO:其他验证用的程序代码要新增在此
	// TODO: Add your control notification handler code here
	params.Save(PARAMFILE);

	CDialog::OnOK();
}
```

**初始化各控制项**

建立`CParamEditDlg::ChangeName`,设定控制项的数值。设定数值需先设定到成员变量后再利用UpdateData。UpdateData的使用方式则是利用调用DoDatExchange的操作,把各值从成员变量设定到控制项(或反向设定)。

DoDataExchange其实就是向导利用变量和控制项ID做为参数,去调用一些函数。这样设定之后,就不需要担心控制项ID的部分会不好处理。

**新增**

在登录新参数时能单击Add按钮。“Add”按钮会调用OnAdd,OnAdd则产生CAddDlg后,利用对话框输入名称。

CAddDlg则是利用“对话框编辑器”产生对话框,再用“类向导”设定跟变量的对应关系。

从参数表格看不到新增的部分,所以所有变量均需设为“1”。

**设定**

为了怕编辑操作有误,当单击“Set”按钮时要能写入表格。单击“Set”按钮后,利用UpdateData从“控制项”把值读到“成员变量”,然后再调用`CFileParam::Add`进行登录。

或许内容不像VisualBasic那么丰富,毕竟对话框的编程量不大,至少可以做到这个程度。

不过它的功能比较弱,例如对话框的形式就不能做缩到最小的处理等等,但是在内部工具和分配的部分已经缠绰有余了。

####执行程序实例

别着急喘口气,先赶快来执行参数编辑器。程序实例就在附赠光盘的“chapter8”文件夹之下。

从光盘复制到硬盘时,应该连参数的数据文件也会一起复制进来,所以一开始有些参数就已经登录进去了,如图8-8所示。

这样就能产生或编辑参数,不过光是这样并没有什么意义。总是要到游戏本身能利用参数编辑器所编辑的数据文件,忙了这么久才有意义嘛!

要是没有先设计编辑工具,又怎么去产生参数数据文件呢?所以才会先从编辑器开始做起。

**利用dummy**

在之前的章节中,我们介绍到像这样必须要有编辑工具才能进行的部分,有时可以做成“dummy”。这个时候,我们把参数直接写在“Params.cpp”里面,可是现在有了参数编辑器,这个程序代码也就无用武之地了。