# Windows ビルド環境 構築手順

Windows インストーラ（`gstreamer-1.0` パッケージ）をビルドする環境の構築手順。

社内の正本は Notion の「Windows MSVC Gstreamerビルド方法」
（技術リソース、自社プロダクト情報 / 手順書）。本ファイルはそれを土台に、
2026-09-03 の実ビルドで判明した事項を反映したもの。

| 項目 | 値 |
| --- | --- |
| ターゲット | `msvc_x86_64` |
| Config | `config/win64.cbc` |
| リポジトリ | `git@github.com:future-standard/cerbero-fs-custom.git`（`fscustom/1.28.5`） |
| OS | Windows 11 Pro / Windows Server 2022 |
| `cerbero bootstrap` 所要時間 | 約 2 時間以上 |
| フルビルド所要時間 | 約 1 時間 45 分（上流CIのタイムアウト値） |
| 成果物 | `gstreamer-1.0-msvc-x86_64-<version>.exe`（Inno Setup、約 843 MB） |

> **リポジトリ移行済み。** Notion の手順書は GitLab の
> `futurestandard/windows-software-decoder/cerbero-gstreamer-windows-build` を指しているが、
> 現行は上記 GitHub リポジトリ。Notion 側のリンクは古い。

---

## 1. マシン仕様

cerbero のフルビルドはソースツリー・ビルドツリー・prefix を同時に抱えるため、検証用VMの標準的な
割り当てでは足りない。ディスクが一番シビア。

| 項目 | 推奨 | 理由 |
| --- | --- | --- |
| ディスク | 200 GB | `C:\gst` 配下（sources / build / dist / logs）だけで 80〜100 GB。加えて Windows + VS BuildTools で 30 GB 前後。可変VHDXで確保する |
| vCPU | 8 コア以上 | ninja の `-j` がそのまま効く。4コアだと現行環境（16スレッド）比で数倍の時間になる |
| メモリ | 16 GB 以上 | gst-plugins-bad の C++ リンク時が最も食う。並列度を上げるほど必要 |
| OS | Win11 Pro / Server 2022 | 上流CIは Windows Server 2022 を使用。Pro は Hyper-V ゲストとして扱いやすい |

---

## 2. 環境準備

### 2-1. システムロケールを UTF-8 にする（重要）

Notion 手順書に「日本語環境だとビルドエラーが発生するため」として記載されている設定。
**設定 → 時刻と言語 → 言語と地域 → 管理用の言語の設定 → システムロケールの変更 →
「ベータ: ワールドワイド言語サポートで Unicode UTF-8 を使用」にチェック**（再起動が必要）。

これを入れないと、コンソールが cp932 のままになり、**ビルドエラーの本文が読めなくなる**。
2026-09-03 の実例:

- gst-plugins-bad が停止したがコンソールには `ninja: build stopped` しか出ず、
  真の原因（`LNK1181: 入力ファイル 'd3dx12-format-properties.lib' を開けません`）が表示されなかった
- meson が cp932 でエラー文字列を出力しようとして `UnicodeEncodeError` を起こし、
  エラーメッセージ自体を握り潰していた（`mesonbuild/scripts/meson_exe.py:78`）
- ログファイルも文字化けし、読むたびに `iconv -f cp932 -t utf-8` が必要だった

#### より影響の小さい代替案: `VSLANG=1033`

システムロケールの変更は全アプリケーションに効くため、副作用を避けたい場合の代替手段。
上の失敗は「リンカの日本語出力がパイプで分断されて U+FFFD に化け、それを cp932 の
stdout に `print()` して例外になる」という経路なので、**MSVC の診断メッセージを英語に
固定すれば出力が純 ASCII になり、経路そのものが成立しなくなる。**

```powershell
[Environment]::SetEnvironmentVariable('VSLANG', '1033', 'User')

# 併用推奨: print() を UTF-8 でエンコードさせ、万一化けても握り潰されないようにする
[Environment]::SetEnvironmentVariable('PYTHONUTF8', '1', 'User')
```

影響範囲は MSVC ツールチェーンと Python プロセスのみ。再起動も不要で、外すのは変数を
消すだけ。副次的な利点として、ログの文字化けも解消するため `iconv -f cp932` での復号が
不要になり、エラーが英語になるので検索もしやすくなる。

**未確認事項:** Notion 手順書がロケール設定で回避しようとしているエラーが、ここで特定した
1 件と同じものかは確認できていない（スクリーンショットのみで本文に記載が無いため）。別の
症状も含む場合は `VSLANG` だけでは足りない可能性がある。まず `VSLANG` を入れてビルドし、
問題が出なければシステムロケールは触らない、という順序が現実的。

### 2-2. 開発者モードをオンにする

設定 → システム → 開発者向け → 開発者モード。Notion 手順書に記載あり。

### 2-3. Smart App Control をオフにする

クリーンインストール直後の Windows 11 は Smart App Control が有効なことがある。有効なままだと、
ローカルでコンパイルされた実行ファイル（cargo の build script、meson のコンパイラ疎通確認
バイナリ、cerbero が自前ビルドした ninja / cargo-c など）が実行時にブロックされ、
`os error 4551`「アプリケーション制御ポリシーによってこのファイルがブロックされました」で
ビルドが random に失敗する。

- Windows セキュリティ → アプリとブラウザーの制御 → Smart App Control → オフ
- ビルド 26100.8246 / 26200.8246 以降（KB5083769, 2026-04-14）なら後からオンに戻せる
- 状態確認: `Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\CI\Policy" -Name VerifiedAndReputablePolicyState`（0=オフ / 1=有効 / 2=評価モード）
- 併せて `CiTool --list-policies` で、IT配布のカスタム WDAC ポリシーが無いことも確認できる
  （2026-09-03 時点の現行機では、適用中のポリシーはすべて Microsoft 標準のもので、
  ユーザーモードの実行を止めうるのは `VerifiedAndReputableDesktop`＝SAC のみだった）

### 2-4. Python

Notion 手順書は **3.11.9 を指定**（「3.12以降だとビルドエラーが出るため」）。
ただし 2026-09-03 の実ビルドは Python 3.14 が入った環境で完走している。
cerbero は `C:\gst\build-tools\bin\python.exe` を自前で持つため、システム側の
バージョン依存は薄い可能性がある。**新環境では 3.11.9 に合わせるのが安全**だが、
新しい版でも通るなら問題視しなくてよい。

---

## 3. セットアップ手順

### 3-1. リポジトリの取得

```powershell
git clone git@github.com:future-standard/cerbero-fs-custom.git
cd cerbero-fs-custom
git checkout fscustom/1.28.5
```

Git が未導入なら先に手動で入れるか、この手順を 3-2 の後に回す。

### 3-2. ビルドツール一式の導入

Chocolatey 経由で必要なものが入る。**管理者権限の PowerShell** が必須
（スクリプト冒頭に `#Requires -RunAsAdministrator`）。

```powershell
Set-ExecutionPolicy -ExecutionPolicy Unrestricted
.\tools\bootstrap-windows.ps1 -VSVersion '2022'
```

導入されるもの:

| 対象 | バージョン / 備考 |
| --- | --- |
| Visual Studio Build Tools | 2022（`Workload.VCTools` + ARM64 ツール + includeRecommended） |
| MSYS2 | `C:\msys64` に固定。winpty を pacman で追加し、`ucrt64.ini` に `MSYS2_PATH_TYPE=inherit` を追記 |
| CMake | 3.31.7（PATH に追加） |
| Git / git-lfs | `/NoAutoCrlf` 付きで導入 |
| Python 3 | 3.9 以上 |
| WiX Toolset | 5.0.1（.NET 8 ランタイム同梱）— MSI 生成用 |
| vcredist140 | — |

> **Notion 手順書の備忘:** 「bootstrap で Visual Studio は自動でインストールされるが、
> Visual Studio Installer で下記を追加しないとビルドが通らなかった」との記載があり、
> スクリーンショットで追加コンポーネントが示されている。
> 具体的なコンポーネント名は Notion のスクリーンショットを参照のこと（本ファイルには未転記）。

### 3-3. MSYS2 に `patch` を追加（自動化されていない）

`patch` はどちらの bootstrap にも含まれていない。`cerbero/bootstrap/windows.py:152` の pacman
パッケージ一覧（flex / bison / gperf / make / diffutils / perl ×2 / ninja）にも、
`bootstrap-windows.ps1` にも無い（grep 済み）。無いまま進むと recipe のパッチ適用ステップで
失敗する。

```sh
# MSYS2 UCRT64 シェルで
pacman -Sy --needed patch
```

上流CIがこれを踏まないのは `CERBERO_BOOTSTRAP_SYSTEM: "no"` でイメージ側に焼き込んでいるため。
恒久対処するなら `MSYS2Bootstrapper.packages` に `'patch'` を追加するコミットを積むのが筋。

### 3-4. cerbero の bootstrap

ツールチェーンと依存物のダウンロード。README いわく「Windows では数時間かかることもある」。
ここは一度きり。

```powershell
.\cerbero-uninstalled -c config/win64.cbc bootstrap
```

`cerbero-uninstalled.ps1` は内部で
`msys2_shell.cmd -ucrt64 -defterm -no-start -here -use-full-path` を呼ぶので、
MSYS2 UCRT64 シェルに入る必要はない。

### 3-5. ビルド

```powershell
.\cerbero-uninstalled -c config/win64.cbc package gstreamer-1.0
```

`config/win64.cbc` の中身は `target_arch = Architecture.X86_64` のみで、64bit Windows では
既定と同じ（`-c` を省いても `msvc_x86_64` になる）。ただし手順書・上流CIと揃えて明示する。

variant の指定は不要。`cerbero/config.py:400` が Windows で variant 無指定なら
`visualstudio` を自動選択するため、そのまま MSVC ビルドになる。

成果物はリポジトリ直下に出る:

```
gstreamer-1.0-msvc-x86_64-1.28.5.exe
```

---

## 4. この手順書の検証状況

| 項目 | 状態 | 根拠 |
| --- | --- | --- |
| 手順 3-1〜3-5 でインストーラが生成されること | 確認済 | 2026-09-03、現行機で完走（104/104 recipe、843 MB の .exe を生成） |
| `patch` の欠落 | 確認済 | grep で bootstrap 全体に不在。実際に失敗を再現 |
| SAC が一連のブロックの原因であること | 確認済 | `VerifiedAndReputablePolicyState: 1`、`CiTool --list-policies` で他に該当ポリシー無し。オフ後に全て通過 |
| `msvc_x86_64` が既定になること | 確認済 | `cerbero/config.py:400` および実出力 |
| `-c config/win64.cbc` の有無で結果が変わらないこと | 確認済 | ファイル内容と実出力の一致 |
| UTF-8 ロケール設定の効果 | **未検証** | 未適用の状態で cp932 由来のエラー隠蔽が起きたことは確認済。設定後に解消するかは未確認 |
| DirectX-Headers の `.lib` 生成 | 確認済 | `recipes/directx-headers.recipe` の `post_install` に実装あり。ただし現行 `dist` の当該ファイルは手動生成のままで、recipe 経由での再生成は未実施 |
| クリーンなマシンでの通し実行 | **未検証** | 上記は既存環境での実績。まっさらな環境からの通しは未実施 |
| 推奨マシン仕様の妥当性 | 推定 | 現行環境のディスク使用量とCI設定からの見積り |

初回の構築時は、失敗した箇所とその対処をこの表に追記していくこと。

---

## 5. 上流CIとの対応

この構成は上流 cerbero の `.gitlab-ci.yml` にある `.cerbero windows native` ジョブと同じ。
迷ったときはこれが正解の定義。

| CI の設定 | 値 |
| --- | --- |
| イメージ | `registry.freedesktop.org/gstreamer/gstreamer/amd64/windows:2026-01-31.0-1.28` |
| ランナー tag | `docker`, `windows`, `gstreamer-windows`, `2022` |
| CONFIG | `win64.cbc` |
| ARCH | `msvc_x86_64` |
| CERBERO_HOST_DIR | `C:/cerbero` |
| CERBERO_BOOTSTRAP_SYSTEM | `no`（システム依存物はイメージに焼き込み済み） |
| 成果物 | `*.msi`, `*.whl`, `*/logs` |
| タイムアウト | 1h45m |

CI は Windows コンテナで動かしているので、Docker for Windows が使える環境なら、このイメージを
そのままローカルで回す選択肢もある（上流と完全に同一の環境が得られる）。

---

## 6. 2026-09-03 のビルドで踏んだ問題

現行機（日常業務用ラップトップ）で発生したもの。大半は「Smart App Control」と
「6月から積み上がった古いビルドツリー」に起因し、クリーンな新環境では発生しない見込み。

| 問題 | 原因 | 新環境での扱い |
| --- | --- | --- |
| ninja.exe / cargo-c 系 / build script / meson sanity check がブロックされる（os error 4551） | Smart App Control | 手順 2-3 でオフにすれば発生しない。バイナリ差し替えは不要 |
| perl が exit code 0xC0E90002 でサイレントクラッシュ | SAC が `perl544.dll` のロードをブロック | 同上。Strawberry Perl での回避は不要 |
| `gst_video_dmabuf_pool_get_type` が未解決（g-ir-scanner の LNK2001） | `dist` に 6月インストール分の古い `gstvideo-1.0.lib` が残っており、g-ir-scanner がそれを掴んだ | クリーンな prefix では起きない。既存ツリーを再利用するなら `dist` を消してから |
| `d3dx12-format-properties.lib` が開けない（LNK1181） | DirectX-Headers の静的ライブラリが `lib*.a` 命名のみで `dist/lib` に入っていた | **対処済**。`recipes/directx-headers.recipe` の `post_install` が MSVC 命名のコピーを作る（`01bd8688`, 2025-05-23）。6月の `dist` にそれが無かったのは、当時のビルドが移行前のリポジトリで行われたためと思われる。現行リポジトリで `directx-headers` を再ビルドすれば正しく生成される |
| 上記エラーがコンソールに表示されない | cp932 環境で meson が `UnicodeEncodeError` | 手順 2-1 で解消する見込み |
| recipe のパッチ適用が失敗 | `patch` が bootstrap に含まれていない | 手順 3-3 で対処 |
| `gstva-1.0` の import library が見つからない警告 | fork の `files_libs_fsvd`（`recipes/gst-plugins-bad-1.0.recipe:120`）が Linux 前提（VA-API）でプラットフォーム条件なし | Windows ビルドでは無害。気になるなら recipe に条件を追加 |
| `qt5` / `qt6` / `vs-templates` / `dvd` パッケージが empty の警告 | variant で無効化されているため中身が無い | 無害 |

---

## 7. このforkの独自差分

`origin/1.28` からの差分は 6 コミット。それ以外は上流そのもの。

| SHA | 内容 |
| --- | --- |
| `06d7da79` | Stop shipping the base system, and the 27 libraries nobody loads |
| `1a440001` | Package the plugins one by one, and the libraries they link |
| `4b9cba31` | Add a package holding only what the FS-VD1000 runs |
| `7879cddc` | mpeg2: fix interlaced MPEG-2 handling (buffer flags + interlace-mode) |
| `7e6b33d5` | soundtouch: update stale tarball_checksum |
| `e42e19eb` | Integrate gstreamer-custom-build patches directly into recipes |

上位3つは FS-VD1000（Linux アプライアンス）向けの絞り込みパッケージ `mlitdecoder-1.0` を
追加するもの。`install_dir` が `Platform.LINUX: /opt/gstreamer-1.0/` のみのため、
Windows ビルドとは別系統（`./cerbero-uninstalled package mlitdecoder-1.0`）。

---

## 8. 検討事項：Windows版の絞り込みパッケージ

stock の `gstreamer-1.0` は 264 プラグインを全部ビルドする。librsvg・accesskit・libproxy で
消耗するのは、これらが stock パッケージに含まれるため。

`mlitdecoder-1.0-runtime.package` のコメントが Linux 側について書いていることは、そのまま
Windows 側にも当てはまる:

> The appliance uses about 30 of them. Every extra one is two liabilities at once:
> a parser or a codec exposed to whatever arrives on the network … and a licence obligation.

Windows インストーラの実際の必要プラグインが同様に限られるなら、`fsvd` グループと同じ考え方で
Windows 用の絞り込みパッケージを定義すれば、ビルド対象そのものが減る。VA / KMS / ALSA の代わりに
d3d11・d3d12・mediafoundation・wasapi を並べる形になる。ビルド時間・CVE 追跡範囲・
ライセンス義務が同時に小さくなる。
