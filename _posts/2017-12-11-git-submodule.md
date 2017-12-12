---
title: Git submodule 的使用
tags: [git, git-submodule]
category: [Tech]
---

因為公司專案的關係接觸到 git submodule 功能，但一直覺得使用上有些疑惑，不論是 command line 還是 GUI Client。這兩天重新看了這篇[文章](https://git-scm.com/book/en/v2/Getting-Help-Submodules)之後，發現原來我將一些在公司上專案的操作行為誤以為是 git submodule 的原生行為了。以下大概整理一下 git submodule 操作方式和幾個問題。

# 在專案中加入 submodule

加入 submodule 的方式依下面兩種場景而有所不同。這邊假設有兩個 repo 分別為 `MyProject` 和 `MySubmodule`，而 `MySubmodule` 將是 `MyProject` 當中的一個 submodule。

## 在本機已存在的 repo 當中

這裡的場景會是你本身有一個專案，想要引入一個已經存在在 git server 上的另一個 repo 當作 submodule。以上面的例子來說就是你本身有一個 `MyProject` repo，然後想把遠端另一個 `MySubmodule` 引入作為 submodule。

操作的方式比較單純，只要在 `MyProject` repo 當中，先進入想放置 submodule 資料夾的目錄 (下面假設為根目錄)，然後下

```
git submodule add [MySubmodule git URL]
```

就行了。在同一目錄下執行 `ls` 可以看到

```
MyProject/
  .git
  .gitmodules
  MySubmodule/
```

你會發現本地端多了 `MySubmodule` repo 的資料夾和另一個 `.gitmodule` 檔；這個檔是 git 用來記錄你 submodule 資訊的地方，主要包含路徑和遠端 repo 的 git URL。

## 直接 clone 一個含有 submodule 的 repo

即直接 clone 一個遠端的 `MyProject` repo，而 `MyProject` 本身就含有 `MySubmodule` 這個 submodule。

不過當你 clone 完畢之後，進入 `MySubmodule` 資料夾會發現是空的。你必須再執行 `git submodule init` 和 `git submodule update` 兩個動作之後，才能得到 `MySubmodule` 的最新內容。

# 更新 submodule

加入 submodule 其實很單純，不過真正讓我混淆的是之後的更新動作。總歸來說，必須要搞清楚 submodule 在主 repo 當中的被記錄方式，並不是直接將 submodule repo 的 git history 拷貝過來，而是以一個類似指標性質的 submodule link 存在的。submodule 的遠端 repo 如果有其它新的 commit，並不會直接影響主 repo 當中的 submodule link (這就是我混淆的點，因為我遇到專案有另外寫 script 來將 submodule 的更新自動上傳到其它 repo，而我一直以為這是 submodule 的原生功能...)。

<!--
舉例來說，我們以 Repo A 和 Repo B 分別代表主 repo 和將被引入作為 submodule 的 repo，並用 Repo A->B 代表存在在 Repo A 當中的 submodule B。

假設 Repo B 本身的最新提交為 `b111111`，則當 Repo A 首次引入 Repo B 的當下，Repo A->B 的最新提交也會指向 `b111111`。這時，如果 Repo B 有人推上新的提交版號 `b2222222`，那本機的 Repo A->B 和遠端的 Repo A->B 會怎麼改變呢?

答案是不會改變。就像先前說的，遠端 Repo A->B 只是記錄當前指向 Repo B 版號的 link，如果不去更新它，是不會改變的。若要更新 Repo A->B，可以進入包含 Repo A->B 的目錄下執行 `git pull`，接著你會發現 Repo A 的 submodule B 目錄成為 unstaged 狀態。當你同樣進行 `git add`, `git commit` 和 `git push` 後，遠端的 Repo A->B 指向的版號才會跟著更新。
-->

如果要更新主 repo 的 submodule link，有兩種方法。其一是進入 submodule 目錄，執行 `git pull`，然後回到主 repo 目錄底下，先利用 `git status` 查看，你會看到 submodule 目錄呈現 unstaged 狀態 (這是因為 `git pull` 之後改變了 submodule link)，將它 commit 並 push 到主 repo 的遠端 URL。

另一種方法是將 submodule 直接 push 到主 repo 的遠端 URL 當中相對應的位置，然後在本機的主 repo 用 `git submodule update` 來更新。

## 實例說明
假設我們已在本地端有 `MyProject` repo，想把 `MySubmodule` 這個 repo 引入作為 submodule，而 `MySubmodule` 裡面目前只有一個關於 README 的 commit。

```
7a04500 Add readme.md
```

在剛執行 `git submodule add` 將 `MySubmodule` 加成submodule。加完後，先在 `MyProject` 目錄 (不要進到 `MySubmodule` 裡面) 執行 `git status`，會顯示

```
On branch master
Your branch is up-to-date with 'origin/master'.
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	new file:   .gitmodules
	new file:   MySubmodule
```

可以先將這個改動 commit。接著用 `git submodule status` 查看，會得到類似下面的訊息

```
7a04500a72c135bde7adc62cff97ce9eccffa290 MySubmodule (heads/master)
```

然後進入 `MySubmodule` 目錄查看 `git log --online`，:

```
7a04500 Add readme.md
```

可以看到 `MySubmodule` 當中的唯一提交。接著，假設 `MySubmodule` 遠端 repo 有別的提交 (05f2ba3)，先在本機 `git fetch` 看看:

```
From gitlab.com:nightsnow0918/MySubmodule
   7a04500..05f2ba3  master     -> origin/master
```

表示目前 `MySubmodule` 遠端最新的版號為 `05f2ba3`。

接下來把這個新的提交 pull 到本機端，到 `MySubmodule` 目錄底下執行 `git pull`。這時切換到 `MyProject` 目錄執行 `git status`，會發現有 unstaged file:

```
	modified:   MySubmodule (new commits)
```

這看起來挺正常的，因為 `MySubmodule` 內容變了嘛，所以 `MyProject` 顯示 `MySubmodule` 為 unstaged 也很合理啊!

基本上算是說對了一半，我們可以用 `git diff` 看一下比較結果:

```
diff --git a/MySubmodule b/MySubmodule
index 7a04500..05f2ba3 160000
--- a/MySubmodule
+++ b/MySubmodule
@@ -1 +1 @@
-Subproject commit 7a04500a72c135bde7adc62cff97ce9eccffa290
+Subproject commit 05f2ba3c25e60ccd386a44bf5eedac6d3798d621
```

有看到最下面兩行嗎? 它顯示的並不是 `README.md` 被改動的內容，而是兩個版號，正好就是更新 `MySubmodule` 前後的兩個版號。由此可知，當 submodule 有了新的提交，對於 `MyProject` 來說僅是 submodule 的「指向版號」變了，`MyProject` 並不會記錄 submodule 內部到底改了什麼。當然，遠端的 `MyProject` repo 也是同樣的道理，你應該常在 GitHub 或 GitLab 看見 submodule 的目錄後面會接著 "@" 和一串文字，那串文字就代表目前這個 submodule 是指向原 repo 的哪個版號。

那如果有其它人更新了 `MyProject` 當中的 submodule，我要怎麼更新到本機呢? 基本上 `git pull` 就能達到你的目的，當然如果你只是想更新 submodule，但不想更新其它部份，也可以下 `git submodule update`。

`git submodule update` 使用上需要注意一點，它是會直接將遠端 repo 的 submodule 指向覆蓋到你本機的狀態。所以如果你本機的 submodule 有自己的改動，執行 `git submodule update` 將會遺失你的改動。
{: .notice--primary}
