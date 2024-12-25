---
created: <% tp.file.creation_date() %>
tags: [timestamp]
---

# <% moment(tp.file.title,'YYYY-MM-DD').format("dddd, MMMM DD, YYYY") %>

<< [[Timestamps/<% tp.date.now("YYYY", -1) %>/<% tp.date.now("MM-MMMM", -1) %>/<% tp.date.now("YYYY-MM-DD-dddd", -1) %>|Yesterday]] | [[Timestamps/<% tp.date.now("YYYY", 1) %>/<% tp.date.now("MM-MMMM", 1) %>/<% tp.date.now("YYYY-MM-DD-dddd", 1) %>|Tomorrow]] >>

---
## :LiBookmarkCheck: Missions

:LiKeyboard: thesis progress

:LiWallet: $

:LiGamepad2: 

- [ ] :LiGlassWater: / 2000ml :LiMilk:

- [ ] :LiBicepsFlexed: / 60 mins

- [ ] :LiCodeXml: daily algorithm question / 1 

- [ ] :LiArrowDownRightFromSquare: daily interview question / 3

---
## :LiNotebookPen: Notes

#八股

---
### Notes Created Today
```dataview
List FROM "" WHERE file.cday = date("<%tp.date.now("YYYY-MM-DD")%>") SORT file.ctime asc
```

### Notes Last Touched Today
```dataview
List FROM "" WHERE file.mday = date("<%tp.date.now("YYYY-MM-DD")%>") SORT file.mtime asc
```