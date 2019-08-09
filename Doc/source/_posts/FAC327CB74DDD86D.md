---
title: 执行程序实例
date: 2019-05-27 15:50:33
tags: [战斗]
comment: false
categories: 战斗
createtime: 1558944931
description: 整理到这里,其实已经可以开始运行看看,不过我们还是先把光盘里的程序实例稍微再加工一下。关于追加部分的处理,先执行程序实例会比较容易懂吧!请实际运行一下程序实例,程序实例就在随书附赠光盘的“chapter6”文件夹下。
---
###执行程序实例

整理到这里,其实已经可以开始运行看看,不过我们还是先把光盘里的程序实例稍微再加工一下。

关于追加部分的处理,先执行程序实例会比较容易懂吧!请实际运行一下程序实例,程序实例就在随书附赠光盘的“chapter6”文件夹下。

####显示状态

首先,把眼标光标对到人物所在的方格上,就会跳出弹出式状态说明。如果不想办法显示出状态的话,游戏玩家不会知道还剩下多少HP值,所以这个功能不能少。这里的程序实例是当光标移动到方格上时即跳出相关说明的方式。

程序实例还有另一个状态说明,只是用来给游戏玩家看,显示经验值不,如图6-9所示。

对程序设计师而言,在固定位置显示状态会比较好写,可是这样会连可显示数量也都固定不能改变。话虽如此,如果只考虑到游戏玩家这边的状态显示,其实数量固定也没什么问题才对。

状态显示是先在内存上产生图像,然后在画面上合成出最后我们所看到的样子。

产生状态图像(Battle.cpp)

```C++
//
// 建立人物状态的图像
//
void CBattleAction::SetPlayerStatus(int player)
{
	Status[player].Copy(&StatusBase);
	if (StatusCharacter[player] != 0) {
		CStatus &status = StatusCharacter[player]->status;
		CImageDC dc(Status + player);
		HFONT	oldFont = dc.SelectObject(hFont);
		dc.SetTextColor(WhitePixel);
		dc.SetBkMode(TRANSPARENT);

		char str[256];
		dc.ExtTextOut(10, 4, 0, 0, status.name, strlen(status.name), NULL);
		dc.ExtTextOut(10, 22, 0, 0, str,
                    sprintf(str, "等级　　　%7d", status.level), NULL);
		dc.ExtTextOut(10, 40, 0, 0, str,
                    sprintf(str, "经验值　　%7d", status.experience), NULL);
		dc.ExtTextOut(10, 58, 0, 0, str,
                    sprintf(str, "HP值　　%3d/%3d",
                        status.hit_point, status.max_hit_point), NULL);
		dc.ExtTextOut(10, 76, 0, 0, str,
                    sprintf(str, "MP值　　%3d/%3d",
                        status.magic_point, status.max_magic_point), NULL);

		dc.SelectObject(oldFont);
	}
}
```

画面下方给游戏玩家看的状态显示都是“文字”显示。因为用DIBSection所产生的图像可以取得DC,故可利用ExtTextOut绘制出文字内容。

#####设计弹出式状态说明

弹出式状态说明(pop-upstatus)其实就是CG的组合而已。虽然做起来颇费工夫,但会比较好应用。

即使设计成等到最后再转成CG、显示到画面上,但是中间只要稍有变动,CG和程序代码两边都得修改,实在很麻烦。因此一开始先“暂时”不要用CG,只用程序去绘制,等到显示内容确定之后再改成CG显示也是一个聪明的做法。

先如图6-10制作出显示用的“背景”CG和“零件”CG,再利用程序组合起来使用。

弹出式状态说明(Battle.cpp)

```C++
//
// 显示突现式状态说明
//
void CBattleAction::ChangeStatus(CPoint point)
{
	if (MapRect.PtInRect(point) && map_data[point.y][point.x].type & (PLAYER | ENEMY)) {
		if (StatusChar != (CCharacter *)map_data[point.y][point.x].sprite) {
			StatusChar = 0;

			RedrawSprite(&PopupSprite);		// 删除之前的显示
			StatusChar = (CCharacter *)map_data[point.y][point.x].sprite;
			CStatus &status = StatusChar->status;

			// 组合状态显示的零件
			PopupImage.Copy(&PopupImageBase);	// 复制原始图像
			PopupImage.DrawText(hFont, 1, 1, status.name, RGB(0, 0, 0));
			PopupImage.DrawText(hFont, 0, 0, status.name);

			int hp_len = status.hit_point * 76 / status.max_hit_point;
			int mp_len = status.magic_point * 76 / status.max_magic_point;
			PopupImage.Copy(&PopupParts, CRect(21, 17, 21 + hp_len, 20), CPoint(0, 20));
			PopupImage.Copy(&PopupParts, CRect(21, 32, 21 + mp_len, 35), CPoint(0, 23));

			char hp_str[4], mp_str[4], mhp_str[4], mmp_str[4];
			sprintf(hp_str, "%03d", status.hit_point);
			sprintf(mp_str, "%03d", status.magic_point);
			sprintf(mhp_str, "%03d", status.max_hit_point);
			sprintf(mmp_str, "%03d", status.max_magic_point);

			for (int i=0; i<3; i++) {
				PopupImage.Copy(&PopupParts,
									CRect(42 + i * 8, 21, 50 + i * 8, 31),
									CPoint((hp_str[i] - '0') * 8, 0));
				PopupImage.Copy(&PopupParts,
									CRect(74 + i * 8, 21, 82 + i * 8, 31),
									CPoint((mhp_str[i] - '0') * 8, 0));
				PopupImage.Copy(&PopupParts, 
									CRect(42 + i * 8, 36, 50 + i * 8, 46),
									CPoint((mp_str[i] - '0') * 8, 10));
				PopupImage.Copy(&PopupParts,
									CRect(74 + i * 8, 36, 82 + i * 8, 46),
									CPoint((mmp_str[i] - '0') * 8, 10));
			}
			point = IndexToPoint(point);
			point.x += (MAPGRID_WIDTH - PopupImage.Width()) / 2;
			point.y -= 64;

			// 若不在范围内，则放在窗口内
			if (point.x < 0) {
				point.x = 0;
			}
			else if (point.x >= WindowWidth - PopupImage.Width()) {
				point.x = WindowWidth - PopupImage.Width();
			}
			if (point.y < 0) {
				point.y = 0;
			}

			PopupSprite.SetDrawPos(point);
			RedrawSprite(&PopupSprite);		// 显示
		}
	}
	else if (StatusChar != 0) {		// 有显示状态，但不在人物上
		StatusChar = 0;
		RedrawSprite(&PopupSprite);		// 删除之前的显示
	}
}
```

HP、MP要先计算显示长度,再从零件去复制需要的长度。

```C++
int hp_len = status.hit_point * 76 / status.max_hit_point;
PopupImage.Copy(&PopupParts, 
				CRect(21, 17, 21 + hp_len, 20),	//复制~到
				CPoint(0, 20));					//从~复制
```

数字转换成字符串后,再逐一复制字符。

```C++
sprintf(hp_str, "%03d", status.hit_point);
for (int i=0; i<3; i++) {
	PopupImage.Copy(&PopupParts,
					CRect(42 + i * 8, 21, 50 + i * 8, 31),
					CPoint((hp_str[i] - '0') * 8, 0));
｝
```

利用这种方法组合显示图像后,让它能配合光标位置显示在画面上。

执行程序,把鼠标光标移动到人物上方,就可以看到光标所指人物的状态说明了。

####输入命令

当鼠标光标移动到人物上并按下鼠标左键时,就会显示菜单。

这个菜单是利用sprite所绘制而成。sprite是从图6-11的菜单用CG而来,由左而右的状态分别是“一般”、“已选用过”和“无法选取”。

只要利用此CG上当时可用命令的部分,即可产生并显示sprite。

显示命令菜单(Battle.cpp)

```C++
// 显示命令菜单
CPoint CBattleAction::CommandIndex(int idx)
{
	// 移动
	switch (idx) {
	  case COMMAND_MOVE:		// 移动
		return CPoint(SelChar->move_done? 140: 0, 0);

	  case COMMAND_ATTACK:		// 直接攻击
		return CPoint(SelChar->attack_done? 140: 0, 22);

	  default:					// 魔法
		idx -= COMMAND_MAGIC_FIRST;
		if (SelChar->IsMagicOk(idx)) {
			return CPoint(SelChar->attack_done? 140: 0,
				SelChar->GetMagicParam(idx).id * 22 + 44);
		}
		return CPoint(-1, -1);
	}
}

void CBattleAction::ShowCommand(CPoint point)
{
	if (point.x > 320)
		point.x -= 70 + 24;
	else
		point.x += 24;

	status = STATUS_COMMAND;
	select = COMMAND_NO_SELECT;

	for (int i=0; i<MAX_COMMAND; i++) {
		CPoint src = CommandIndex(i);
		if (src.x >= 0) {
			CommandSprite[i].SetDrawPos(point.x, point.y);
			CommandSprite[i].SetSrcPos(src.x, src.y);
			CommandSprite[i].Show();
			RedrawSprite(CommandSprite + i);
			point.y += 23;
		}
	}
}
```

因为菜单是配合鼠标光标位置而显示,所以当光标在画面右边时,菜单应显示在左方,当光标在画面左边时,则菜单应显示在右方,这样菜单才不会跑到画面之外。

选择命令菜单(Battle.cpp)

```C++
// 选择命令菜单--让它反白
void CBattleAction::HighlightCommand(int newsel, bool selection)
{
	if (newsel != COMMAND_NO_SELECT) {
		CSprite *sp = CommandSprite + newsel;
		sp->SetSrcPos(selection? 70: 0, sp->GetSrcPos().y);
		RedrawSprite(sp);
	}
}

void CBattleAction::HighlightCommand(CPoint point)
{
	int newsel = FindCommand(point);
	if (select != newsel) {
		HighlightCommand(select, false);
		select = newsel;
		HighlightCommand(select, true);
	}
}
```

当鼠标光标放在菜单上时,则应做“反白显示”。反白就是CG的第2个(已选择),因此错开sprite的显示CG就可以切换已选择/解除选择。

删除命令菜单(Battle.cpp)

```C++
// 删除命令菜单
void CBattleAction::HideCommand()
{
	for (int i=0; i<MAX_COMMAND; i++) {
		if (CommandSprite[i].IsShow()) {
			CommandSprite[i].Show(false);
			RedrawSprite(CommandSprite + i);
		}
	}
}
```

因为菜单是sprite显示,故将sprite设为“不显示”即可删除菜单。

执行命令(Battle.cpp)

```C++
// 选择命令菜单
void CBattleAction::SelectCommand(CPoint point)
{
	int sel = FindCommand(point);
	switch (sel) {
	  case COMMAND_NO_SELECT:
		break;

	  case COMMAND_MOVE:
		HideCommand();
		status = STATUS_MOVE;
		FindDist(SelPos.x, SelPos.y, SelChar->status.move_dist);
		break;

	  case COMMAND_ATTACK:
		HideCommand();
		status = STATUS_ATTACK;
		FindAttack(SelPos.x, SelPos.y, SelChar->status.attack_dist);
		break;

	  default:
		HideCommand();
		magic_param = SelChar->GetMagicParam(sel - COMMAND_MAGIC_FIRST);
		if (magic_param.type == MAGIC_TYPE_SELF) {
			status = STATUS_MAGIC_PLAYER;
			FindAttack(SelPos.x, SelPos.y, magic_param.dist, false);
		}
		else {
			status = STATUS_MAGIC_ENEMY;
			FindAttack(SelPos.x, SelPos.y, magic_param.dist);
		}
		break;
	}
}

// 从命令菜单的位置开始搜寻
int CBattleAction::FindCommand(CPoint point)
{
	if (!SelChar->move_done && CommandSprite[COMMAND_MOVE].PtIn(point))
		return COMMAND_MOVE;

	if (!SelChar->attack_done) {
		for (int i=COMMAND_ATTACK; i<MAX_COMMAND; i++) {
			if (CommandSprite[i].PtIn(point))
				return i;
		}
	}
	return COMMAND_NO_SELECT;
}
```

若已选择命令时,则移往“下一个步骤”。下一个步骤是指,如为移动则显示移动范围(如为攻击则显示攻击范围),然后等待游戏玩家点选。

等到游戏玩家所选人物的行动均已设定之后,再按鼠标右键选择“Turnend”(本轮结束),就换成轮到敌方动作。“Turnend”菜单也是以script显示。敌方的动作会根据前面所说明的计算方式进行。

如此重复下去就是一场战斗了。

因为游戏结束的故事大纲还没写,所以指在VisualC++的调试窗口显示“打赢”或“战败”来表示游戏结束,不过输赢只在是否全军覆没,相信各位不需要去看VisualC++的调试窗口就知输赢如何了。

还有,这里并没有用到随机数,因此是游戏玩家胜出的参数。如果怎么打都打不赢,稍微提高游戏玩家这边的HP就能让敌方兵败如山倒。嗯,各位也可以考虑在调试用的部分放一个不为人知的“必胜方程序”的命令(别搞错,这只是调试用而已)。