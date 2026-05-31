---
title: "管理者権限なしのWindowsにPrefectを入れてみた"
emoji: "🔧"
type: "tech"
topics: ["prefect", "python", "windows", "workflow"]
published: true
---

会社のPCなので管理者権限がない。それでも副業のパイプライン管理を改善したくて、Prefectを入れようとした記録です。

## なぜPrefectを入れようとしたか

自作のPythonスクリプトが20本以上になってきて、Windowsのタスクスケジューラで管理していたんだけど、どのスクリプトがいつ動いてどこで止まったか把握しにくくなってきた。

Prefectを使えばUIで実行履歴を見られるし、失敗したときのアラートも出せる。ただ「管理者権限なしで入るのか？」というのが最初の疑問だった。

## やってみたこと

まず普通にインストールしてみた。

```
pip install prefect
```

エラーは出なかった。インストール自体はできる。

ただ `prefect version` を叩いたら「prefect は認識されないコマンドです」と出た。

原因はすぐわかった。`pip install` にデフォルトで `--user` フラグが付いていて、インストール先が `AppData\Roaming\Python\Python313\Scripts` になっていた。ここがPATHに入っていなかった。

管理者権限があればシステム全体のPATHを変えられるけど、ユーザーPATHなら自分でいじれる。PowerShellで確認したら案の定そのパスが入っていなかった。

## 解決方法

ユーザーPATHに追加するだけ。

```powershell
$prefectPath = "C:\Users\ユーザー名\AppData\Roaming\Python\Python313\Scripts"
$userPath = [Environment]::GetEnvironmentVariable("PATH", "User")
[Environment]::SetEnvironmentVariable("PATH", "$userPath;$prefectPath", "User")
```

ターミナルを再起動したら `prefect version` が通った。

```
Version:              3.7.2
Python version:       3.13.12
Server type:          ephemeral
```

## 管理者権限なしで入れるときのポイント

- `pip install prefect` でOK（`--user` は自動で付く）
- PATHは `[Environment]::SetEnvironmentVariable` でユーザースコープに追加
- `C:\Program Files` 系には書けないが `AppData` 以下なら問題ない

今のところPrefect自体は動いている。次はローカルで簡単なフローを書いて、既存のスクリプトと組み合わせていく予定。

---

ちなみに自分はソフトウェアのQAエンジニアで、副業でAI関連のスクリプトをいくつか動かしている。LangGraphとかPrefectとかのOSSは完全に独学なので、わかったことをそのまま書いていく。
