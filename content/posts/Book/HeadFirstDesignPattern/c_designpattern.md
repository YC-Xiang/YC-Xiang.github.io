https://www.cnblogs.com/Chary/p/11736497.html

# 观察者模式

```c++
struct object
{
	struct observer* observer_list[MAX_BINDING_NUMBER];
	int number;
  
	void (*notify)(struct object* object);
	void (*add_observer)(struct observer* observer);
	void (*del_observer)(struct observer* observer);
}

struct observer
{
	struct object* object;

	void (*update)(struct observer* observer);
};

void bind_observer_to_object(struct observer* observer, struct object* object)
{
	observer->object = object;
	object->add_observer(observer);
}
  
void unbind_observer_from_object(struct observer* observer, struct object* object)
{
	object->del_observer(observer* observer);
	memset(observer, 0, sizeof(struct observer));
}

void notify(struct object* object)
{
	struct observer* observer;

	for(int index = 0; index < pObject->number; index++) {
        	observer = object->observerList[index];
        	observer->update(observer);
	}
}
```

# 装饰器模式

```c++
struct object
{
    struct object* prev;
  
    void (*decorate)(struct object* object);
}

void decorate(struct object* object)  
{
	object->prev->decorate(object->prev);
	printf("normal decorate!\n");
}
```
