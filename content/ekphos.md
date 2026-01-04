# ekphos
1. `R` Seems not work well
2. `ci...` might be impRovVE? Such as `ci(` even when the cursor not in the (),instead,go to the nearest (....).
3. `code` have the nice looking,why not give the `**` a nice one? 
4. The relative wikilink path might be useful.In obsidian, we can choose relative path and obsolute path, and when we choose obsolute path,the editor will automatically fill the filename like `path/to/file/name | name`. And this only show the filename without whole path.Maybe the `|` can be considered?
5. Will line number and relative number be integrated? It's useful!
6. I found that the vim "verb" and "noun" are likely separated? For example, when input `dt(`, it goes `d` and `t(`, so the cursor go but doesn't delete, then input `d`, the `dd` goes, it may be a bit odd.
7. I found that the vim "verb" and "noun" are likely separated? For example, when input `dt(`, it goes `d` and `t(`, so the cursor go but doesn't delete, then input `d`, the `dd` goes, it may be a bit odd.
t t //
jjj
111
111
Hello 这是一个中文测试
-   **`R` (Replace Mode)**: The `R` command doesn't seem to function correctly or as expected.
mand 
[[测试]]
[[t1st/test2/test2]]
[[test/测试]]
[[_测-试/test]]
-   **example, executing `ci(` should ideally jump to and change the content of the **nearest** pair of parentheses, even if the cursor is currently outside of them.
[[00_日志/2025-11-26]]
[[test/test]]
[[01_知识库/数据结构/0-1背包问题]]
[[01_知识库/杂项/archlinux疑难杂症]]
- -   **WikiLink Paths**: It would be helpful to support both relative and absolute paths for WikiLinks, similar to Obsidian. For absolute paths, the editor could automatically format them as `[[path/to/file|filename]]` to keep the display clean (showing only the filename).

-   **Hybrid Line Numbers**: Will there be integration for **Hybrid Line Numbers** (showing the absolute number for the current line and relative numbers for others)? It’s a huge boost for navigation.

-   **Vim Grammar Consistency**: I’ve noticed some inconsistencies with "verbs" and "nouns." For instance, when I type `dt(`, it seems to only move the cursor without deleting. I then have to press `d` again (resulting in `dd`) to delete. The action should be atomic.
- `yG` seems not work.`yw` is ok.
- 执行`.` 有问题，执行了`dw`以后，再执行`.`却还是执行`dd`
- `u` 的撤回逻辑好像有问题，在进行`dd`删除行以后，按u并不能恢复，会和下一行连在一起，好像是末尾的回车没了？
- 宏录制有问题，`qa` then `I//<esc>j`,then `@a`,only into insert mode,but no code output.
[[Home]]