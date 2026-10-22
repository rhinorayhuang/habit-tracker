# habit-tracker

互相督促的習慣打卡 App：和朋友組成小組，打卡、看彼此的連續天數，到了設定的時間還沒打卡就提醒。

這也是我往 Architect 前進的第一個練習專案，重點在**做技術決策並記錄取捨**。

## 結構

```
web/            Vue PWA 前端
api/            Go 後端（modular monolith）
docs/
  plan.md       12 週計畫
  adr/          架構決策紀錄（ADR）
  journal/      每週週記
```

## 文件

- [12 週計畫](docs/plan.md)
- [架構決策紀錄](docs/adr/)：新的 ADR 從 [template.md](docs/adr/template.md) 複製
- [週記](docs/journal/)：新的週記從 [template.md](docs/journal/template.md) 複製

## 在兩台電腦之間同步

第一次（在自己的電腦，用個人 GitHub 帳號）：

```bash
git remote add origin https://github.com/<你的帳號>/habit-tracker.git
git push -u origin main
```

之後每次：

- **離開這台電腦前**：`git add`、`git commit`、`git push`
- **到另一台電腦後，開始做事前**：`git pull`
