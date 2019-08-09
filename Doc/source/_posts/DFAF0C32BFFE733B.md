---
title: 播放音效
date: 2019-05-30 14:10:38
tags: [其他]
comment: false
categories: 大功告成
createtime: 1559197432
description: 游戏进行中有没有音效所造成的第一印象绝对有差别,这里是利用Wave文件和MCI来播放音效。虽然MCI没有多重音效的效果,但只要选对地方,就算是单音也能收到画龙点睛之效。
---
###播放音效

游戏进行中有没有音效所造成的第一印象绝对有差别,这里是利用Wave文件和MCI来播放音效。虽然MCI没有多重音效的效果,但只要选对地方,就算是单音也能收到画龙点睛之效。

播放声音的是`CMainWin::StartWave`函数,它在“攻击”、“魔法”和“提升等级”时会有音效。

在“攻击”等的设定场景已经有动画效果,所以音效是在动画开始时就播放。

攻击时的音效(Battle.cpp)

```C++
void CBattleAction::Attack(CCharacter *from, CCharacter *to, bool player)
{
	...

	// 攻击的动画开始
	AttackFrom = from;
	AttackTo = to;
	AttackPlayerToEnemy = player;
	AttackAnimeCount = 0;

	status = STATUS_ANIME;
	Parent->SetTimer(TIMER_ATTACK_ID, TIMER_ATTACK_TIME);

	// 播放音效
	Parent->StartWave(ATTACK_WAVE);
}
```

开始播放后,StartWave会回到程序,在背景音乐中继续播放。也可配合动画,“让人物随着音效而动作”。

除了战斗时的音效外,事件部分也有音效。事件的音效是使用“sound”命令。

事件的音效(start.txt)

```C++
;
; 游戏结束
;
*gameover

load ov gameover		; 显示GAMEOVER
update wipe

sound A5_06179			; 游戏结束的音效
```

程序实例的故事大纲若有设定在游戏结束时会播放音效,音效会继续播放直到故事大纲停止进行下去。

这里的处理也是从CScriptAction调用出`CMainWin::StartWave`之后,再播放Wave。故事大纲的编写方式是等到播放完毕,但程序则要写成接到播放完毕的通知。

以MCI播放时是指定“notify”,故结束播放时会传送MMMCINOTIFY消息到窗口去。这个消息则是由`CMainWin::OnMciNotify`函数来处理。

MCI的消息处理(MainWin.cpp)

```C++
//
// MCI发出的事件
//
// 因为设定是用play来做call back，所以在播放结束时调用
//
void CMainWin::OnMciNotify(unsigned flag, unsigned id)
{
	if (flag == MCI_NOTIFY_SUCCESSFUL) {
		if (music && music->GetId() == id) {
			Action->MusicDone(MusicNo);
		}
		else if (wave.GetId() == id) {
			wave.Stop();
			Action->WaveDone();
		}
	}
}
```

这里是若为CD播放结束时,则调用`CAction::MusicDone`,如为Wave播放结束,则调用`CAction::WaveDone`。

对啦,就是用这个WaveDone才知道要结束音效。

***
声音素材
***

承蒙日本“株式会社Data Craft”热情赞助,惊慨同意本书的音效使用该公司“声音辞典VOL.5RPG&恐怖/游戏音效”中所收录的“声音素材”。

自己制作、录制音效不是一件简单任务,利用这类相关素材精选集也是一种解决办法。

不过,有些素材精选集中的素材不是播放时间过长,过短,就是音量过大,有时不能直接拿来用。这时候,可以利用加工Wave档的工具软件(如)果只是音量的问题,Windows系统内建的“录音机”就能处理),从高价位的录音专用多功能软件,到免费软件,这类工具软件多得令人眼花擦乱,而且通常声霸卡也多半会附赠简易的声音编辑软件。想利用这种工具软件时,能找到各种“基本声音”的素材集真的很好用。

相信大家都很在意版权问题,“可以用在自制的游戏软件上吗?”是大多数人急欲了解的疑问,请放心,以本书所提到的“声音辞典”为例,该产品声明“只要是已注册的读者,无论营利或自用均可使用”,因此应该可以应用到自制软件上(详细声明内容请参见该产品的“使用注意事项”)。