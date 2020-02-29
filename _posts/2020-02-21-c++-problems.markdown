#### [C++ unordered_map（哈希表）](https://www.jianshu.com/p/56bb01df8ac7)




#### [滑动窗口：两个指针](https://leetcode-cn.com/problems/find-all-anagrams-in-a-string/solution/hua-dong-chuang-kou-tong-yong-si-xiang-jie-jue-zi-/)
        vector<int> findAnagrams(string s, string p) {
                unordered_map<char, int> window;
                unordered_map<char, int> needs;
                vector<int> res;
                int match=0;
                for(char c:p) needs[c]++;
                int left=0,right=0;
                while(right<s.size()){
                    char c1=s[right];
                    if(needs.count(c1)) {
                        window[c1]++;
                        if(window[c1]==needs[c1]) match++;
                    }
                    right++;
                    while(match==needs.size()){
                        char c2=s[left];
                        if(right-left==p.size()) res.push_back(left);
                        if(needs.count(c2)) {
                            window[c2]--;
                            if(window[c2]<needs[c2]) match--;
                        }
                        left++;
                    }
                }
                return res;
            }

只需要根据要求更改res就可以解决一类问题。




#### [c++accumulate函数，计算数组的和](http://c.biancheng.net/view/682.html)




#### [sort函数，第三个参数如果比较复杂的正确表达形式](http://c.biancheng.net/view/561.html)




#### [字符串的操作函数](https://blog.csdn.net/tzheng2008/article/details/7342562)




#### [C++内联函数](https://www.runoob.com/cplusplus/cpp-inline-functions.html)
在类定义中的定义的函数都是内联函数，即使没有使用 inline 说明符。




#### volatile
volatile 关键字是一种类型修饰符，用它声明的类型变量表示可以被某些编译器未知的因素（操作系统、硬件、其它线程等）更改。所以使用 volatile 告诉编译器不应对这样的对象进行优化。
volatile 关键字声明的变量，每次访问时都必须从内存中取出值（没有被 volatile 修饰的变量，可能由于编译器的优化，从 CPU 寄存器中取值）
const 可以是 volatile （如只读的状态寄存器）
指针可以是 volatile




#### C 库宏 - assert()
expression -- 这可以是一个变量或任何 C 表达式。如果 expression 为 TRUE，assert() 不执行任何动作。如果 expression 为 FALSE，assert() 会在标准错误 stderr 上显示错误消息，并中止程序执行。




#### [pragma pack(n)](https://baike.baidu.com/item/%23pragma%20pack)
设定结构体、联合以及类成员变量以 n 字节方式对齐
如果申请的空间<n的大小，就要补全




#### [extern "C"](https://baike.baidu.com/item/extern%20%22C%22/15267013)
extern "C" 的作用是让 C++ 编译器将 extern "C" 声明的代码当作 C 语言代码处理，可以避免 C++ 因符号修饰导致代码不能和C语言库中的符号进行链接的问题。


#### 编写若干个整数（以0为结束），逆序存入一个单向链表中

        #include <stdio.h>
        typedef struct node {
                int val;
                struct node *next;
        }*nodeptr;
        void main() {
                nodeptr head = (nodeptr)malloc(sizeof(nodeptr));
                nodeptr p;
                int number;
                head->next = NULL;
                do {
                        scanf("%d", &number);
                        p = (nodeptr)malloc(sizeof(nodeptr));
                        p->next = head->next;
                        head->next = p;
                        p->val = number;
                } while (number != 0);
                while (p != NULL) {
                        printf("%d", p->val);
                        p = p->next;
                }
        }


