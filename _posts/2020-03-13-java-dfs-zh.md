---
layout    : post
title     : "水壶问题-盲目搜索问题"
date      : 2020-03-13
lastupdate: 2020-03-13
categories: 数据结构 dfs
---

 **人工智能课的一个小练习，直接DFS暴力搜索即可。**

问题背景：给定两个水壶，一个可装4加仑水，一个能装3加仑水。水壶上没有任何度量标记。有一水龙头可用来往壶中灌水。问题是怎样在能装4加仑的水壶里恰好只装2加仑水。

这个问题的求解方法已在教材中给出。下面只是给出程序实现。

 

//对于水壶问题的定义的几个操作：

//0. 　空操作

//1.　4加仑壶不满时将其加满

//2.　3加仑壶不满时将其加满

//3.　把4加仑水壶中的水全部倒出

//4.　把3加仑水壶中的水全部倒出

//5.  把3加仑水壶中的水往4加仑水壶中倒，直到4加仑水壶满

//6.  把4加仑水壶中的水往3加仑水壶中倒，直到3加仑水壶满

//7.  把3加仑水壶中的水全部倒往4加仑水壶中

//8. 　 把4加仑水壶中的水全部倒往3加仑水壶中

 

//解决的基本思路是利用状态空间法求解，设X为4加仑水壶的水量，Y是3加仑水壶的水量。

//则所有可能的(X，Y)序对作为状态空间M的状态集，操作{1, 2，3, 4，5, 6，7, 8}是输入字符集。

//开始状态是(0,0)，终止状态是{(2，Y)}，Y可能是0，1，2，3。

//当一个输入串是M的语言时，这个串就是水壶问题的一个可能方案。

 

//编程的考虑是：Water[5][4]标识数组，为1时，表示这个状态已经到达，为0时表示这个状态没有到达。

**运行效果截图：**
![在这里插入图片描述](/assets/img/2020-03-13-java-dfs-zh/1.png)


**java代码：**

```java
public class 水壶问题 {
    static int maxX=4,maxY=3;
    static int[][] water=new int[maxX+1][maxY+1];
    static LinkedList<Output> path=new LinkedList<>();
    static int resCount=1;
    public static void main(String[] args) {
        jiashui(0,0);
    }
    public static void jiashui(int x,int y){
        if (x==2){
            System.out.println("第"+(resCount++)+"种解决方案:");
            System.out.println("4加仑水壶\t3加仑水壶\t所使用的操作");
            path.forEach(System.out::println);
            return;
        }
        if (water[0][0]==0){
            water[0][0]=1;
            path.add(new Output(0,0,0));
            jiashui(0,0);
            path.removeLast();
            water[0][0]=0;
        }
        if (x<maxX && water[maxX][y]==0){
            water[maxX][y]=1;
            path.add(new Output(maxX,y,1));
            jiashui(maxX,y);
            path.removeLast();
            water[maxX][y]=0;
        }
        if (y<maxY && water[x][maxY]==0){
            water[x][maxY]=1;
            path.add(new Output(x,maxY,2));
            jiashui(x,maxY);
            path.removeLast();
            water[x][maxY]=0;
        }
        if (x>0 && water[0][y]==0){
            water[0][y]=1;
            path.add(new Output(0,y,3));
            jiashui(0,y);
            path.removeLast();
            water[0][y]=0;
        }
        if (y>0 && water[x][0]==0){
            water[x][0]=1;
            path.add(new Output(x,0,4));
            jiashui(x,0);
            path.removeLast();
            water[x][0]=0;
        }
        if (x+y>=maxX &&water[maxX][x+y-maxX]==0){
            water[maxX][x+y-maxX]=1;
            path.add(new Output(maxX,x+y-maxX,5));
            jiashui(maxX,x+y-maxX);
            path.removeLast();
            water[maxX][x+y-maxX]=0;
        }
        if (x+y>=maxY &&water[x+y-maxY][maxY]==0){
            water[x+y-maxY][maxY]=1;
            path.add(new Output(x+y-maxY,maxY,6));
            jiashui(x+y-maxY,maxY);
            path.removeLast();
            water[x+y-maxY][maxY]=0;
        }
        if(x+y<=maxX && y>0&& water[x+y][0]==0){
            water[x+y][0]=1;
            path.add(new Output(x+y,0,7));
            jiashui(x+y,0);
            path.removeLast();
            water[x+y][0]=0;
        }
        if(x+y<=maxY && x>0&&water[0][x+y]==0){
            water[0][x+y]=1;
            path.add(new Output(0,x+y,8));
            jiashui(0,x+y);
            path.removeLast();
            water[0][x+y]=0;
        }


    }
    private static class Output{
        int x;
        int y;
        int opt;
        public Output(int x,int y,int opt){
            this.x=x;
            this.y=y;
            this.opt=opt;
        }

        @Override
        public String toString() {
            return "Output{" +
                    "x=" + x +
                    ", y=" + y +
                    ", opt=" + opt +
                    '}';
        }
    }
}
```