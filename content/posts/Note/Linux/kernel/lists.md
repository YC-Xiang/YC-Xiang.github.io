双向循环链表 `list.h`, 列出了一些 driver 常用的 list operations:

```c++
#define LIST_HEAD_INIT(name)
#define LIST_HEAD(name)

void INIT_LIST_HEAD(struct list_head *list)

void list_add(struct list_head *new, struct list_head *head) // 在 head 之后插入 list node
void list_add_tail(struct list_head *new, struct list_head *head) // 在 head 之前插入 list node
void list_del(struct list_head *entry) // 删除当前 list node
void list_del_init(struct list_head *entry);
void list_move(struct list_head *list, struct list_head *head);
void list_move_tail(struct list_head *list, struct list_head *head);
int list_is_first(const struct list_head *list, const struct list_head *head)
int list_is_last(const struct list_head *list, const struct list_head *head)
int list_empty(const struct list_head *head)
int list_is_singular(const struct list_head *head)
void list_splice(const struct list_head *list, struct list_head *head)
void list_splice_tail(struct list_head *list, struct list_head *head)
void list_splice_init(struct list_head *list, struct list_head *head)
void list_splice_tail_init(struct list_head *list, struct list_head *head)
#define list_entry(ptr, type, member)
#define list_first_entry(ptr, type, member)
#define list_last_entry(ptr, type, member)
#define list_first_entry_or_null(ptr, type, member)
#define list_next_entry(pos, member)
#define list_prev_entry(pos, member)
#define list_for_each(pos, head)
#define list_for_each_safe(pos, n, head) // 如果在遍历的时候会删除当前节点，那么需要使用 xxx_safe, 如果只是读 list 可以使用 list_for_each
#define list_for_each_entry(pos, head, member)
#define list_for_each_entry_reverse(pos, head, member)
#define list_for_each_entry_continue(pos, head, member)
#define list_for_each_entry_continue_reverse(pos, head, member)
#define list_for_each_entry_from(pos, head, member)
#define list_for_each_entry_safe(pos, n, head, member)
#define list_for_each_entry_safe_reverse(pos, n, head, member)
```
