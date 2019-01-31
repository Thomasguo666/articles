# fhq Treap <span id="1">

- **[fhq Treap](#1)**
  - **[原理](#1.1)**
  - **[基本操作](#1.2)**
    - **[split](#1.2.1)**
    - **[merge](#1.2.2)**
    - **[del](#1.2.3)**
    - **[insert](#1.2.4)**
    - **[rank](#1.2.5)**
    - **[kth](#1.2.6)**
    - **[pre/pst](#1.2.7)**
    - **[各种区间操作](#1.2.8)**
  - **[可持久化](#1.3)**
  - **[垃圾回收](#1.4)**
  - **[最终代码](#1.5)**

# 

我们发现，普通平衡树Treap不支持区间操作，常用来实现文艺平衡树的Splay由于在splay的过程中形态发生了变化，故不适用于可持久化，而且平衡树的旋转写起来都很讨厌。那么，有没有一种平衡树不用旋转，支持区间操作，而且还可以可持久化呢？

fhq Treap正好可以达到这一点。

前置技能：普通平衡树

约定：

ch\[x][0/1]表示x的左/右孩子

val[x]表示x的权值，dat\[x]表示x的关键码（既然是Treap，就要有关键码。）

sz[x]表示x的子树大小（包括x）

tag[x]表示x的标记（含义待会再讲）



## 原理 <span id="1.1">

fhq Treap的原理十分简单粗暴，就是把要操作的区间扯下来，打个标记再拼回去。

## 基本操作 <span id="1.2">

### #1 split <span id="1.2.1">

格式：split(now,k,&x,&y);

split操作分两种，第一种是权值分裂，第二种是排名分裂。

权值分裂就是把now的子树中权值小于等于k的节点放在x上，其余的节点放在y上

举个西瓜：

![](https://cdn.luogu.org/upload/pic/49411.png)



如果我们进行split(3,1,x,y)，那么得到的结果就是

x=1,y=3

![](https://cdn.luogu.org/upload/pic/49419.png)（对不起，split操作并不能帮你把节点的边框加粗）

这怎么实现呢

```c++
void split(int now,int k,int &x,int &y)
{
	if (!now) 
    {
        x=y=0; //如果now不存在，那么分裂成的两棵子树也不存在
        return ;
    }
    if (tag[now]) down(now); //先不管他
    if (val[now]<=k) //如果now的权值比k小，那么权值比k大的节点一定只在now的右子树（bst性质），所以分裂的地方在now的右子树
	{
		x=now;//x的根节点就是now。
		split(ch[now][1],k,ch[now][1],y);//把now的右子树中权值比k小的部分当作now的右子树，其余的部分放在y。
		update(now); //辅助操作，即更新now的信息
	}
	else //否则分裂的地方在now的左子树
	{
		y=now;//y的根节点就是now
		split(ch[now][0],k,x,ch[now][0]);//同上
		update(now);
	}
    //为什么不把两个update(now)一起写在这里呢？是为了方便可持久化。
}
```

这样下来，now的子树就被分成了x，y两部分。

排名分裂就是把now的子树（包括now）的中序遍历中前k个节点放在x，其余的节点放在y

实现跟权值分裂差不多。代码：

```c++
void split(int now,int k,int &x,int &y)
{
	if (!now) {x=y=0;return;}
	if (tag[now]) down(now);
	if (sz[ch[now][0]]<k)
	{
		x=now;
		split(ch[now][1],k-sz[ch[now][0]]-1,ch[now][1],y);
		update(now);
	}
	else 
	{
		y=now;
		split(ch[now][0],k,x,ch[now][0]);
		update(now);
	}
}
```

### #2 merge <span id="1.2.2">

格式：merge(x,y)

就是把x，y两个树合并在一起。

详情看代码：

```cpp
int merge(int x,int y)
{ 
	if (!x||!y) return x+y; //如果两个子树有一个为空，那么就返回非空的那一个。
	if (dat[x]<dat[y]) 
	{
		if (tag[x]) down(x);//别管他
		ch[x][1]=merge(ch[x][1],y); //为了满足堆性质，使树的高度尽量小
		update(x);
		return x;
	}
	else
	{
		if (tag[y]) down(y);
		ch[y][0]=merge(x,ch[y][0]);
		update(y);
		return y;
	}
}
```



### #3 del <span id="1.2.3">

格式：del(a)

就是把权值为a的节点删除一个

怎么删呢？

把这个节点切掉，再把断开的两部分粘起来

代码

```c++
void del(int a)
{
	split(root,a,x,z);//这里的split应该是权值分裂（显然）
	split(x,a-1,x,y);
	y=merge(ch[y][0],ch[y][1]);//如果有重复节点
	root=merge(merge(x,y),z);
}
```

如果没有重复节点，就这样写

```c++
void del(int a)
{
	split(root,a,x,z);//这里的split应该是权值分裂（显然）
	split(x,a-1,x,y);
	root=merge(x,z);
}
```



### #4 insert <span id="1.2.4">

格式：insert(a）

如图：

![](https://cdn.luogu.org/upload/pic/49442.png)(如果有重复权值的话，<a应该为$\leq$a)

代码

```c++
int new_node(int a)//新建节点
{
	sz[++tot]=1,val[tot]=a,dat[tot]=rand();
	return tot;
}
void insert(int a)
{
	split(root,a,x,y);//把小于等于a的节点放在x，大于a的节点放在y，用权值分裂
	root=merge(merge(x,new_node(a)),y);把x，a，和y拼起来
}
```

### #5 rank <span id="1.2.5">

格式：rank(a)

查询a的排名。

把小于a的节点数量加1就是a的排名了

代码

```c++
int rank(int a)
{
	split(root,a-1,x,y);
	int ans=sz[x]+1;
	merge(x,y);
	return ans;
}
```

### #6 kth <span id="1.2.6">

格式：kth(now,k)

查询now的子树中第k小值所在的节点。

同普通平衡树，需要递归实现

```cpp
int kth(int now,int k)
{
	if (k==sz[ch[now][0]]+1) return now;
	if (k<=sz[ch[now][0]]) return kth(ch[now][0],k);
	else return kth(ch[now][1],k-sz[ch[now][0]]-1);
}
```

### #7  pre/pst <span id="1.2.7">

格式：pre(a)/pst(a)

查询a的前驱后继

实现很简单，看代码应该能懂。

```c++
int pre(int a)
{
	split(root,a-1,x,y);
	int ans=val[kth(x,sz[x])];
	root=merge(x,y);
	return ans;
}
int pst(int a)
{
	split(root,a,x,y);
	int ans=val[kth(y,1)];
	root=merge(x,y);
	return ans;
}

```



### #8 各种区间操作 <span id="1.2.8">

** 注意：如果有区间操作，必须用排名分裂！**

为什么呢？因为一段区间里的数，权值不一定是连续的。

而fhq Treap一个很好的性质，就是中序遍历以后就是原序列。

区间操作怎么实现呢？先看代码：

```c++
void rev(int l,int r)//以区间反转为例
{
	split(root,l-1,x,y);//排名分裂
	split(y,r-l+1,y,z);
    //也可以改成split(root,r,x,z),split(x,l-1,x,y),作用是把区间分成[1,l-1],[l,r],[r+1,n]三段
	tag[y]^=1;//在中间一段打标记
	root=merge(x,merge(y,z));//拼回去
}

```

接下来揭晓down函数的作用：

如果你要反转区间[1,2,3,4]，假设这个区间被保存在y节点上

那么你先在y上打标记。

如果你不需要对y进行split或merge操作，那么无论它有没有被反转你甭管

但是如果你要对它split或merge的话，你就应该先交换它的两个子树，比如说变为[3,4,1,2]。

然后再在[3,4]和[1,2]上打标记（如果已经有标记了，那么取消）。

```c++
 void down(int x)
{
	swap(ch[x][0],ch[x][1]);
	if (ch[x][0]) tag[ch[x][0]]^=1;
	if (ch[x][1]) tag[ch[x][1]]^=1;
	tag[x]=0;
}

```

区间求和:

```c++
ll query(int l,int r)
{
	split(root,l-1,x,y);
	split(y,r-l+1,y,z);
	ll ans=sum[y];
	root=merge(x,merge(y,z));
	return ans;
}

```



## 可持久化 <span id="1.3">

fhq Treap如何可持久化呢？就是在split和merge操作里克隆要操作的节点，然后在克隆的节点（而非原节点）进行操作。

而在本题中，由于每个merge操作之前都会split一遍，所以merge操作里就不用克隆了。

```c++
int clone(int y)
{
    int x=++tot;
	ch[x][0]=ch[y][0],ch[x][1]=ch[y][1],val[x]=val[y],dat[x]=dat[y],sz[x]=sz[y];
    return x;
}
void split(int now,int k,int &x,int &y)
{
	if (!now) {x=y=0;return;}
	if (tag[now]) down(now);
	if (sz[ch[now][0]]<k)
	{
		x=clone(now);
		split(ch[x][1],k-sz[ch[x][0]]-1,ch[x][1],y);
		update(x);
	}
	else 
	{
		y=clone(now);
		split(ch[y][0],k,x,ch[y][0]);
		update(y);
	}
}

```



## 垃圾回收 <span id="1.4">

（不是回收我）

为了~~环保~~节省空间，所以del、clone和new_node操作得改一改

```c++
 int new_node(int x)
{
	int a=cantop?can[cantop--]:++tot;
	sz[a]=1,sum[a]=val[a]=x,dat[a]=rand();
	return a;
}
int clone(int x)
{
	int a=cantop?can[cantop--]:++tot;
	sz[a]=sz[x],sum[a]=sum[x],val[a]=val[x],ch[a][0]=ch[x][0],ch[a][1]=ch[x][1],dat[a]=dat[x],tag[a]=tag[x];
	return a;
}
void del(int &root,int a)//为了可持久化，除了split和merge以外的每一个基本操作都要加上&root参数（必须是引用）
{
	split(root,a,x,z);
	split(x,a-1,x,y);
	can[++cantop]=y;
	root=merge(x,z);
}

```

## 最终代码 <span id="1.5">

[普通平衡树](https://www.luogu.org/problemnew/show/P3369)

```c++
#include <bits/stdc++.h>
using namespace std;
const int N=100005;
int ch[N][5],val[N],dat[N],sz[N],root,tot,x,y,z;
void update(int n)
{
	sz[n]=1+sz[ch[n][0]]+sz[ch[n][1]];
}
void split(int now,int k,int &x,int &y)
{
	if (!now) x=y=0;
	else if (val[now]<=k)
	{
		x=now;
		split(ch[now][1],k,ch[now][1],y);
		update(now);
	}
	else 
	{
		y=now;
		split(ch[now][0],k,x,ch[now][0]);
		update(now);
	}
}
int merge(int x,int y)
{
	if (!x||!y) return x+y;
	if (dat[x]<dat[y])
	{
		ch[x][1]=merge(ch[x][1],y);
		update(x);
		return x;
	}
	else
	{
		ch[y][0]=merge(x,ch[y][0]);
		update(y);
		return y;
	}
}
int new_node(int a)
{
	sz[++tot]=1,val[tot]=a,dat[tot]=rand();
	return tot;
}
void insert(int a)
{
	split(root,a,x,y);
	root=merge(merge(x,new_node(a)),y);
}
void del(int a)
{
	split(root,a,x,z);
	split(x,a-1,x,y);
	y=merge(ch[y][0],ch[y][1]);
	root=merge(merge(x,y),z);
}
int rank(int root,int a)
{
	split(root,a-1,x,y);
	int ans=sz[x]+1;
	merge(x,y);
	return ans;
}
int kth(int root,int k)
{
	if (k==sz[ch[root][0]]+1) return root;
	if (k<=sz[ch[root][0]]) return kth(ch[root][0],k);
	else return kth(ch[root][1],k-sz[ch[root][0]]-1);
}
int pre(int a)
{
	split(root,a-1,x,y);
	int ans=val[kth(x,sz[x])];
	root=merge(x,y);
	return ans;
}
int pst(int a)
{
	split(root,a,x,y);
	int ans=val[kth(y,1)];
	root=merge(x,y);
	return ans;
}
int main()
{
    srand(time(0));
	int n;
    cin>>n;
    while(n--)
    {
        int opt,a;
        cin>>opt>>a;
        if(opt==1) insert(a);
        else if(opt==2) del(a);
        else if(opt==3) cout<<rank(root,a)<<endl;
        else if(opt==4) cout<<val[kth(root,a)]<<endl;
        else if(opt==5) cout<<pre(a)<<endl;
        else cout<<pst(a)<<endl;
    }
    return 0;
}
```

[文艺平衡树](https://www.luogu.org/problemnew/show/P3391)

```c++
#include <bits/stdc++.h>
using namespace std;
const int N=100005;
int ch[N][5],val[N],dat[N],sz[N],tag[N],root,tot,x,y,z;
void read(int &n)
{
	int fh=1,ans;
	char c;
	while (!isdigit(c=getchar())) if (c=='-') fh=-1;
	ans=c-48;
	while (isdigit(c=getchar())) ans=ans*10+c-48;
	n=ans*fh;
}
void down(int x)
{
	swap(ch[x][0],ch[x][1]);
	if (ch[x][0]) tag[ch[x][0]]^=1;
	if (ch[x][1]) tag[ch[x][1]]^=1;
	tag[x]=0;
}
void update(int n)
{
	sz[n]=1+sz[ch[n][0]]+sz[ch[n][1]];
}
void split(int now,int k,int &x,int &y)
{
	if (!now) {x=y=0;return;}
	if (tag[now]) down(now);
	if (sz[ch[now][0]]<k)
	{
		x=now;
		split(ch[now][1],k-sz[ch[now][0]]-1,ch[now][1],y);
	}
	else 
	{
		y=now;
		split(ch[now][0],k,x,ch[now][0]);
	}
	update(now);
}
int merge(int x,int y)
{
	if (!x||!y) return x+y;
	if (dat[x]<dat[y])
	{
		if (tag[x]) down(x);
		ch[x][1]=merge(ch[x][1],y);
		update(x);
		return x;
	}
	else
	{
		if (tag[y]) down(y);
		ch[y][0]=merge(x,ch[y][0]);
		update(y);
		return y;
	}
}
int new_node(int a)
{
	sz[++tot]=1,val[tot]=a,dat[tot]=rand();
	return tot;
}
void rev(int l,int r)
{
	split(root,l-1,x,y);
	split(y,r-l+1,y,z);
	tag[y]^=1;
	root=merge(x,merge(y,z));
}
void print(int now)
{
	if (!now) return;
	if (tag[now]) down(now);
	print(ch[now][0]);
	printf("%d ",val[now]);
	print(ch[now][1]);
}
int main()
{
    srand(time(0));
	int n,m;
	read(n),read(m);
	for (int i=1;i<=n;i++) root=merge(root,new_node(i));
	for (int i=0;i<=n;i++) printf("%d %d %d %d %d\n",ch[i][0],ch[i][1],sz[i],val[i],dat[i]);
	for (int i=1;i<=m;i++)
	{
		int l,r;
		read(l),read(r);
		rev(l,r);
	}
	print(root);
    return 0;
}

```

[可持久化平衡树](https://www.luogu.org/problemnew/show/P3835)

```c++
#include <bits/stdc++.h>
using namespace std;
const int N=500005;
int ch[N<<4][5],val[N<<4],dat[N<<4],sz[N<<4],root[N],tot,x,y,z;
void update(int n)
{
	sz[n]=1+sz[ch[n][0]]+sz[ch[n][1]];
}
void copy(int x,int y)
{
	ch[x][0]=ch[y][0],ch[x][1]=ch[y][1],val[x]=val[y],dat[x]=dat[y],sz[x]=sz[y];
}
void split(int now,int k,int &x,int &y)
{
	if (!now) x=y=0;
	else if (val[now]<=k)
	{
		x=++tot;
		copy(x,now);
		split(ch[x][1],k,ch[x][1],y);
		update(x);
	}
	else 
	{
		y=++tot;
		copy(y,now);
		split(ch[y][0],k,x,ch[y][0]);
		update(y);
	}
}
int merge(int x,int y)
{
	if (!x||!y) return x+y;
	if (dat[x]<dat[y])
	{
		int p=++tot;
		copy(p,x);
		ch[p][1]=merge(ch[p][1],y);
		update(p);
		return p;
	}
	else
	{
		int p=++tot;
		copy(p,y);
		ch[p][0]=merge(x,ch[p][0]);
		update(p);
		return p;
	}
}
int new_node(int a)
{
	sz[++tot]=1,val[tot]=a,dat[tot]=rand();
	return tot;
}
void insert(int &root,int a)
{
	split(root,a,x,y);
	root=merge(merge(x,new_node(a)),y);
}
void del(int &root,int a)
{
	split(root,a,x,z);
	split(x,a-1,x,y);
	y=merge(ch[y][0],ch[y][1]);
	root=merge(merge(x,y),z);
}
int rank(int &root,int a)
{
	split(root,a-1,x,y);
	int ans=sz[x]+1;
	merge(x,y);
	return ans;
}
int kth(int &root,int k)
{
	if (k==sz[ch[root][0]]+1) return root;
	if (k<=sz[ch[root][0]]) return kth(ch[root][0],k);
	else return kth(ch[root][1],k-sz[ch[root][0]]-1);
}
int pre(int &root,int a)
{
	split(root,a-1,x,y);
	if (!x) return -2147483647;
	int ans=val[kth(x,sz[x])];
	root=merge(x,y);
	return ans;
}
int pst(int &root,int a)
{
	split(root,a,x,y);
	if (!y) return 2147483647;
	int ans=val[kth(y,1)];
	root=merge(x,y);
	return ans;
}
int main()
{
    srand(time(0));
	int n;
    cin>>n;
    for (int i=1;i<=n;i++)
    {
        int v,opt,a;
        cin>>v>>opt>>a;
        root[i]=root[v];
        if(opt==1) insert(root[i],a);
        else if(opt==2) del(root[i],a);
        else if(opt==3) cout<<rank(root[i],a)<<endl;
        else if(opt==4) cout<<val[kth(root[i],a)]<<endl;
        else if(opt==5) cout<<pre(root[i],a)<<endl;
        else cout<<pst(root[i],a)<<endl;
    }
    return 0;
}
```

[可持久化文艺平衡树](https://www.luogu.org/problemnew/show/P5055)

```c++
#include <bits/stdc++.h>
using namespace std;
typedef long long ll;
const int N=200005;
int ch[N<<6][3],val[N<<6],dat[N<<6],sz[N<<6],tag[N<<6],root[N],tot,x,y,z;
int can[N],cantop;
ll sum[N<<6];
void read(int &n)
{
	int fh=1,ans;
	char c;
	while (!isdigit(c=getchar())) if (c=='-') fh=-1;
	ans=c-48;
	while (isdigit(c=getchar())) ans=ans*10+c-48;
	n=ans*fh;
}
int new_node(int x)
{
	int a=cantop?can[cantop--]:++tot;
	sz[a]=1,sum[a]=val[a]=x,dat[a]=rand();
	return a;
}
int clone(int x)
{
	int a=cantop?can[cantop--]:++tot;
	sz[a]=sz[x],sum[a]=sum[x],val[a]=val[x],ch[a][0]=ch[x][0],ch[a][1]=ch[x][1],dat[a]=dat[x],tag[a]=tag[x];
	return a;
}
void down(int x)
{
	swap(ch[x][0],ch[x][1]);
	if (ch[x][0]) ch[x][0]=clone(ch[x][0]),tag[ch[x][0]]^=1;
	if (ch[x][1]) ch[x][1]=clone(ch[x][1]),tag[ch[x][1]]^=1;
	tag[x]=0;
}
void update(int x)
{
	sz[x]=1+sz[ch[x][0]]+sz[ch[x][1]];
	sum[x]=val[x]+sum[ch[x][0]]+sum[ch[x][1]];
}
void split(int now,int k,int &x,int &y)
{
	if (!now) {x=y=0;return;}
	if (tag[now]) down(now);
	if (sz[ch[now][0]]<k)
	{
		x=clone(now);
		split(ch[x][1],k-sz[ch[x][0]]-1,ch[x][1],y);
		update(x);
	}
	else 
	{
		y=clone(now);
		split(ch[y][0],k,x,ch[y][0]);
		update(y);
	}
}
int merge(int x,int y)
{
	if (!x||!y) return x+y;
	else if (dat[x]<dat[y])
	{
		if (tag[x]) down(x);
		ch[x][1]=merge(ch[x][1],y);
		update(x);
		return x;
	}
	else
	{
		if (tag[y]) down(y);
		ch[y][0]=merge(x,ch[y][0]);
		update(y);
		return y;
	}
}
void insert(int &root,int k,int a)
{
	split(root,k,x,y);
	root=merge(merge(x,new_node(a)),y);
}
void del(int &root,int a)
{
	split(root,a,x,z);
	split(x,a-1,x,y);
	can[++cantop]=y;
	root=merge(x,z);
}
void rev(int &root,int l,int r)
{
	split(root,l-1,x,y);
	split(y,r-l+1,y,z);
	tag[y]^=1;
	root=merge(x,merge(y,z));
}
ll query(int &root,int l,int r)
{
	split(root,l-1,x,y);
	split(y,r-l+1,y,z);
	ll ans=sum[y];
	root=merge(x,merge(y,z));
	return ans;
}
int main()
{
	int n;
	ll lastans=0;
	read(n);
	for (int i=1;i<=n;i++)
	{
		int v,opt,p,x,l,r;
		read(v),read(opt);
		root[i]=root[v];
		if (opt==1)
		{
			read(p),read(x);
			p^=lastans,x^=lastans;
			insert(root[i],p,x);
		}
		else if (opt==2)
		{
			read(p);
			p^=lastans;
			del(root[i],p);
		}
		else if (opt==3)
		{
			read(l),read(r);
			l^=lastans,r^=lastans;
			if (l>r) swap(l,r);
			rev(root[i],l,r);
		}
		else
		{
			read(l),read(r);
			l^=lastans,r^=lastans;
			if (l>r) swap(l,r);
			lastans=query(root[i],l,r);
			printf("%lld\n",lastans);
		}
	}
	return 0;
}

```

[返回顶部](#home)