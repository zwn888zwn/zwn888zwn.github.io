---
layout    : post
title     : "三子棋AI+java实现+极大极小算法+alphabeta剪枝"
date      : 2020-03-13
lastupdate: 2020-03-13
categories: 数据结构
---

**极大极小算法**的总体思想就是，甲乙双方进行博弈，假设双方都想获得最大收益的情况下，确定甲做出的最优选择。 应用在棋类问题上，就是甲方要考虑自己最大利益时，也要考虑乙方最大利益时的情况。

 - getValue（）用来判断场面局势，把空格全部填上Mark后 满足3个连一起的个数。
 - 对棋盘进行DFS获取对应深度的最大最小值，最后返回到第0层时，选择最大值的走法。

下面这张图帮助理解，我们从0层开始dfs遍历，达到设置深度也就是叶子节点时，用getvalue（）求出当前局势，根据对应层数选择最值。

 - alpha-beta剪枝：假如我们已经遍历完now的左子树，得到beta值为3，又因为其父节点的alpha值大于3，剪枝，注意每次剪枝的比较对象是父节点。（假设now右节点为9，大于左节点，那么min后now还是3，父节点选择max时，不会选到。假设now右节点为1，小于左节点，那么min后now是1，比选来更小，不会选到。）

![在这里插入图片描述](/assets/img/2020-03-13-java-alphabeta-zh/1.png)

**最后附上效果图和代码：**  电脑是O，玩家是X

![在这里插入图片描述](/assets/img/2020-03-13-java-alphabeta-zh/2.png)



```java
package homework;

import java.util.Scanner;

public class 最大最小算法ab剪枝 {
    //3子棋
    final int maxDepth=2;
    final int SIZE=3;
    int[][] arr=new int[SIZE][SIZE];
    int count=0;
    final int MAX_VAL=100,MIN_VAL=-100;


    private class TempNode{
        int val;
        int x;
        int y;
        public TempNode(int val){
            this.val=val;
        }
        public TempNode(){
        }
    }

    public TempNode  dfs(int dp,int alpha,int beta){
        if (dp==maxDepth || count==SIZE*SIZE){
            TempNode node=new TempNode(getValue(arr,dp));
            return node;
        }
        int x=-1,y=-1;
        for (int i=0;i<SIZE;i++) {
            for (int j = 0; j < SIZE; j++) {
                if (arr[i][j]==0){
                    arr[i][j]=dp%2==0?1:2;
                    count++;
                    TempNode node=dfs(dp+1,alpha,beta);
                    //返回时更新ab？
                    if (dp%2==0){//max a 同时记录最值坐标点
                        if (alpha<node.val){
                            alpha=node.val;
                            x=i;
                            y=j;
                        }
                        if (node.val>=beta){ //ab剪枝
                            arr[i][j]=0;
                            return new TempNode(node.val);
                        }


                    }else {//min b
                        if (beta>node.val){
                            beta=node.val;
                            x=i;
                            y=j;
                        }
                        if (node.val<=alpha){
                            arr[i][j]=0;
                            return new TempNode(node.val);
                        }

                    }
                    arr[i][j]=0;
                    count--;

                }
            }
        }
        TempNode node=new TempNode();
        node.x=x;
        node.y=y;
        node.val=dp%2==0?alpha:beta;
        return node;

    }

    /**
     *
     * @param arr
     * @return 把空格全部填上Mark后 满足3个连一起的个数
     */
    private int getValue(int[][] arr,int dp) {
        int mark=dp%2==0?2:1,count=0;

        for (int i=0;i<SIZE;i++){
            if ((arr[i][0]==0 || arr[i][0]==mark ) &&(arr[i][1]==0 || arr[i][1]==mark )&&(arr[i][2]==0 || arr[i][2]==mark )){
                count++;
            }
            if ((arr[0][i]==0 || arr[0][i]==mark ) &&(arr[1][i]==0 || arr[1][i]==mark )&&(arr[2][i]==0 || arr[2][i]==mark )){
                count++;
            }
            if ((arr[i][0]==1 ) &&(arr[i][1]==1 )&&(arr[i][2]==1 ) || (arr[0][i]==1 ) &&(arr[1][i]==1 )&&( arr[2][i]==1 )){
                return 50;//win
            }
            if ((arr[i][0]==2 ) &&(arr[i][1]==2 )&&(arr[i][2]==2 ) || (arr[0][i]==2 ) &&(arr[1][i]==2 )&&( arr[2][i]==2 )){
                return -50;
            }
        }
        //斜
        if (((arr[0][0]==1 ) &&(arr[1][1]==1)&&(arr[2][2]==1))||((arr[0][2]==1) &&(arr[1][1]==1  )&&(arr[2][0]==1))){
            return 50;
        }
        if (((arr[0][0]==1 ) &&(arr[1][1]==1)&&(arr[2][2]==1))||((arr[0][2]==1) &&(arr[1][1]==1  )&&(arr[2][0]==1))){
            return -50;
        }
        if ((arr[0][0]==0 || arr[0][0]==mark ) &&(arr[1][1]==0 || arr[1][1]==mark )&&(arr[2][2]==0 || arr[2][2]==mark )){
            count++;
        }
        if ((arr[0][2]==0 || arr[0][2]==mark ) &&(arr[1][1]==0 || arr[1][1]==mark )&&(arr[2][0]==0 || arr[2][0]==mark )){
            count++;
        }

        return mark==1?count:-count;

    }


    public void print(){
        for (int i=0;i<3;i++){
            for (int j=0;j<3;j++){
                System.out.print("|");
                if (arr[i][j]==0)
                    System.out.print(" ");
                else if (arr[i][j]==1)
                    System.out.print("O");
                else
                    System.out.print("X");
                System.out.print("|");
            }
            System.out.println();
        }
    }

    public static void main(String[] args) {
        最大最小算法ab剪枝 obj=new 最大最小算法ab剪枝();
        obj.arr[1][1]=1;
        obj.count=1;
        obj.print();

        Scanner scan=new Scanner(System.in);
        while (obj.count<9){
            System.out.println("请输入坐标");
            int x=scan.nextInt();
            int y=scan.nextInt();
            obj.count++;
            if (obj.arr[x][y]==0){
                obj.arr[x][y]=2;
                TempNode node=obj.dfs(0,obj.MIN_VAL,obj.MAX_VAL);
                if (obj.arr[node.x][node.y]==0){
                    obj.arr[node.x][node.y]=1;
                    obj.count++;
                    if(obj.getValue(obj.arr,0)>=50){
                        System.out.println("you lose");
                        obj.print();
                        return;
                    }
                }
                else{
                    if (obj.getValue(obj.arr,0)<=-50)
                        System.out.println("you win!");
                    else
                        System.out.println("平局");
                }

                obj.print();
            }else {
                System.out.println("非法输入");
            }


        }

    }

}

```