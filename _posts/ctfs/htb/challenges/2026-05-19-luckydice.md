---
title: HTB「LuckyDice」Easy
description: サイコロの目の総和が，最も大きいプレイヤーを選ぶ．
date: 2026-05-19 22:00:00 +0900
categories: [CyberSecurity, CTF]
tags: [misc, hack_the_box, pwntools]
---

# 20260517-htb-chall-misc-easy-LuckyDice

## Challenge Overview

このプログラムで，`loop()` がtrueになり，`print(flag)` に到達するためには，0.3秒以内にサイコロの目の総和が最も大きい **Player** 番号を入力しなければいけません．
同点の場合は，**最後にサイコロを振ったプレイヤー (プレイヤー番号が大きい方)** が優先されます．
また，ラウンド毎に，`ラウンド数 * 2 + 2` の増加速度でサイコロの数が増加します．

`challenge.py`
```python
#!/usr/bin/env python3
import random
import time
import sys
import os

rounds = 100
player_nr = random.randint(8, 13)
timeout = 0.3

def userinput(question, correct_answer):
    # Show question
    
    print(question)
    # Get user response
    given_answer = input("> ")
    # Check answer
    return (str(correct_answer) == given_answer.strip())

def loop(rnd):
    print("")

    # サイコロ数
    dice_nr = rnd * 2 + 2
    # 結果保存用
    dice_res = {}
    dice_sum = {}

    print('-----------   ' + 'Round ' + str(rnd+1) + '    -----------')
    for player in range(player_nr):
        dice_list = []
        rp = ('round_' + str(rnd+1), 'player_' + str(player+1))
        dice_sum[rp] = 0
        for dice in range(dice_nr):
            rpd = ('round_' + str(rnd+1), 'player_' + str(player+1), 'dice' + str(dice+1))
            dice_res[rpd] = int(random.randint(1, 6))
            dice_list.append(dice_res[rpd])
            dice_sum[rp] += dice_res[rpd]

        time.sleep(0.1)
        print('Player ' + str(player+1) + ': {dices}'.format(dices = ' '.join(map(str, dice_list))))

    result = sorted(dice_sum.items(), key=lambda x:x[1])[-1][0][1].split('_')[1]

    time.sleep(0.05)
    print('Who wins this round?')
    for player in range(player_nr):
        print(str(player+1) + '. Player' + str(player+1))

    start = time.time() 
    answer = input('> ')

    # 入力が完了するまでの経過時間が，0.3秒以下であること
    if time.time() - start > timeout:
        print("Mate... your are too slow! It seems your CPU is too old!")
        return False

    # 勝者の番号ミス
    if answer != result:
        print('Hey! I think your scorring is off... Check your RAM... It may be corrupted!')
        return False

    print('Yes.. Correct!')
    return True


print("WELCOME TO THE DICE ARENA ...")
print("     ____             ")
print("    /\' .\\    _____   ")
print("   /: \\___\\  / .  /\\  ")
print("   \' / . / /____/..\\ ")
print("    \\/___/  \'  '\\  / ")
print("             \'__'\\/  ")
time.sleep(0.3)
print("Welcome my fellow bot!")
time.sleep(0.2)
print("I will need your help!")
time.sleep(0.1)
print("We are taksed to keep the score on this human game.")

time.sleep(0.5)
print("The game is simple!")
time.sleep(0.2)
print("Let's go over the rules...")
time.sleep(0.5)
print("1. On each round, each human player roles several dice.")
time.sleep(0.2)
print("2. The outcome of the dice is added to the player's score.")
time.sleep(0.5)
print("3. The round is won by the player with the highest overall score.")
time.sleep(0.5)
print("4. If there is a draw, the player who rolled the last dice wins the round.")
time.sleep(1) 
print("Simple as that!")
time.sleep(0.5)
print("Now... Clean your memory and prepare your registers ...")


print("The humans are about to start the game! Are you ready?")
print("1. Yes")
print("2. No")
if not input("> ").strip() == "1": # noの場合は即座に終了
    print("Too bad... bye bye")
    sys.exit()

# Start game
print("")
time.sleep(0.5)
print("3 ...")
time.sleep(0.5)
print("2 ...")
time.sleep(0.5)
print("1 ...")
time.sleep(0.5)
print("Go!")

# 100 ラウンド
for i in range(rounds):
    if not loop(i):
        print("Check your system and come back later... bye bye")
        sys.exit()

time.sleep(0.2)
print()
print()
print("Nice job!")
time.sleep(0.2)

print("Here is your prize:")
flag_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), "flag.txt")
with open(flag_path) as f:
    flag = f.read().strip()
print(flag)
```

## Solution

Pwntoolsを使用して自動化します．

`solver.py`
```python
#!/usr/bin/env python3
from pwn import *
import re

# context.log_level = "debug"

p = remote("xxx.xx.xxx.xx", 12345)
# p = process("./challenge.py")

# 特定の文字列を受信した後に，指定した文字列＋改行（`\n`）を送信する
# io.sendlineafter(delim, data)
# - delim (delimiter): 待機する条件となる文字列 (例: b"選択してください: ")
# - data: 送信したいデータ (例: b"1")
p.sendlineafter(b"> ", b"1")

try:
    while True:
        players = []

        while True:
	        # データを1行 (改行コード `\n` が来るまで) 読み込み，それを文字列 (String) として返す関数
            line = p.recvlineS()

            print(line, end="")

            if "Who wins this round?" in line:
                break

            if "Player" not in line:
                continue

            m = re.search(r"Player\s+(\d+):\s+(.+)", line)

            if m is None:
                continue

            player_num = int(m.group(1))

            dice = list(map(int, m.group(2).split()))
            score = sum(dice)

            # score同点なら後のplayerが勝つ
            players.append((score, player_num))

        winner = max(players)[1]

        print(f"Winner = Player {winner}")

        p.sendline(str(winner).encode())

except EOFError:
    print("Process finished")
```