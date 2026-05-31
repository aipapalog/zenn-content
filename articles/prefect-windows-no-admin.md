---
title: "管理者権限なしのWindowsにPrefectを入れてみた"
emoji: "🔧"
type: "tech"
topics: ["prefect", "python", "windows", "workflow"]
published: true
---

管理者権限のないWindows環境にPrefectを導入しようとした記録です。詰まった点と解決方法、タスクスケジューラ・他のOSSとの比較も書きます。

## なぜPrefectを選んだのか

自作のPythonスクリプトが20本以上になってきて、Windowsのタスクスケジューラで管理していた。ただ、これが辛くなってきた。

- どのスクリプトがいつ動いたか一覧で見られない
- 失敗してもメールも通知も来ない
- 依存関係（AのあとにB）を表現できない
- ログが散らばっていてデバッグが面倒

### 他の選択肢と比較した

| ツール | メリット | デメリット |
|---|---|---|
| **タスクスケジューラ（現状）** | 追加インストール不要 | 視認性・通知・依存管理がない |
| **Airflow** | 機能豊富・実績あり | 重い・Windowsは非推奨・管理者権限必要なことが多い |
| **Prefect** | 軽量・ローカル動作・UIあり・Python native | まだメジャーではない |
| **Dagster** | 型安全・テスト重視 | 設定が多い・学習コスト高め |

Airflowは試したがWindows環境ではセットアップが重く断念。PrefectはPython関数にデコレータを付けるだけで動くシンプルさと、ローカルUIで実行履歴を確認できる点が決め手だった。

## インストールで詰まったこと

まず普通にインストールしてみた。

```
pip install prefect
```

エラーは出なかった。インストール自体はできる。

ただ `prefect version` を叩いたら「prefect は認識されないコマンドです」と出た。

原因はすぐわかった。`pip install` にデフォルトで `--user` フラグが付いていて、インストール先が `AppData\Roaming\Python\Python313\Scripts` になっていた。ここがPATHに入っていなかった。

管理者権限があればシステム全体のPATHを変えられるが、ユーザーPATHなら自分で変更できる。PowerShellで確認したら案の定そのパスが入っていなかった。

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

自分はソフトウェアのQAエンジニアで、個人でAI関連のスクリプトをいくつか動かしている。LangGraphとかPrefectとかのOSSは完全に独学なので、わかったことをそのまま書いていく。
