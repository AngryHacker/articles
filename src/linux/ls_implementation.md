# ls 命令的实现

在 linux 下，我们用的最多的命令应该是 ls 吧，那么，你有没有想过这个命令怎么是实现呢？其实了解了 UNIX 环境的相关接口后，也就不难了~


### 目标
可以用 ls 列出目录的简略信息， ls -l 列出目录的详细信息。默认列出本工作目录下的信息，可通过参数指定目录，可同时列出多个目录信息。


### 实现
实现的方式有非常多种，只要能够以良好的风格做出来就行，我下面简单说说自己的思路。

1. 用 is_detail 变量来记录是否列出文件详细信息， initpath 表示将要读取的目录路径。
2. 如果用户没有指定目录，则直接设置路径为当前工作目录，获取目录下所有文件后，进行排序，接着简略输出或者详细输出。结束程序。
3. 用户有提供目录参数的时候，判断是否要进行详细输出，设置 is_detail 变量。
4. 对于每个用户指定目录参数，循环进行：获取目录下所有文件名，进行排序，根据 is_detail 打印信息。

思路是如此的简单，你只要把每一个模块的细节处理好就行了。


主要的辅助函数有：
* getWidth 获取终端的宽度，用于判断简略输出文件名时能放下多少列。
* get_dir_detail 获取指定目录下的所有文件名。
* getPermission 根据文件权限位 st_mode 获取文件权限信息，用字符串表示。
* print_file_info 根据文件指针输出文件信息。
* search_file_info 遍历处理目录下的所有文件。
* init  根据路径是文件还是目录进行分别处理。
* print_simple  只输出文件名，即简略显示


### 代码

```C++
#include <stdio.h>
#include <dirent.h>
#include <stdlib.h>
#include <libgen.h>
#include <sys/stat.h>
#include <string.h>
#include <time.h>
#include <pwd.h>
#include <grp.h>
#include <sys/ioctl.h>
#include <unistd.h>
 
#define MAX_FILE 1000
 
/*保存文件名和文件所在目录*/
typedef struct item{
    char d_name[256];
    char dir_name[256];
}file_item;
 
file_item files[MAX_FILE];
 
/*保存文件信息*/
typedef struct item_info{
    unsigned int i_node;
    char permission[16];
    short owner;
    short group;
    off_t size;
    time_t mod_time;
    nlink_t link_num;
    char name[256];
}info;
 
int is_detail;
int size_of_path;
int terminalWidth;
 
/* 获得用户名 */
const char  *uid_to_name( short uid )
{
	struct	passwd *pw_ptr;
 
	if ( ( pw_ptr = getpwuid( uid ) ) == NULL )
		return "Unknown" ;
	else
		return pw_ptr->pw_name ;
}
 
/* 获得组名 */
const char *gid_to_name( short gid )
{
	struct group *grp_ptr;
 
	if ( ( grp_ptr = getgrgid(gid) ) == NULL )
		return "Unknown" ;
	else
		return grp_ptr->gr_name;
}
 
/* 比较函数 */
int cmp(const void *a, const void *b)
{
    return strcasecmp((*(file_item *)a).d_name,(*(file_item *)b).d_name);
}
 
/*获得终端宽度*/
void getWidth()
{
    struct winsize wbuf;
    terminalWidth = 80;
    if( isatty(STDOUT_FILENO) )
    {
        if(ioctl(STDOUT_FILENO, TIOCGWINSZ, &wbuf) == -1 || wbuf.ws_col == 0)
        {
            char *tp;
            if( tp = getenv("COLUMNS") )
                terminalWidth = atoi( tp );
        }
        else
            terminalWidth = wbuf.ws_col;
    }
    return ;
 }
 
/*获取所有文件名*/
void get_dir_detail(const char* dirname)
{
    DIR* dir;
    struct dirent* drt;
    int cur = 0;
    /*打开目录*/
    dir = opendir(dirname);
    if(dir == NULL)
    {
        perror("Read directroy Error.");
        return;
    }
    /*遍历目录*/
    while((drt = readdir(dir)) != NULL)
    {
        file_item* cur_node = files + cur;
        /*忽略 . 和 .. 目录*/
        if((strcmp(drt->d_name,".")==0)||(strcmp(drt->d_name,"..")==0))
            continue;
 
        if(drt->d_name[0] == '.')
            continue;
 
        size_of_path ++;
        cur++;
        if(cur >= MAX_FILE)
        {
            printf("Too many files!\n");
            exit(-1);
        }
 
        /*文件名*/
        strcpy(cur_node->d_name,drt->d_name);
        /*目录名*/
        strcpy(cur_node->dir_name,dirname);
        strcat(cur_node->dir_name,drt->d_name);
    }
    closedir(dir);
}
 
/*获取文件权限*/
void getPermission(mode_t st_mode, char* permission)
{
    /*初始化 111000000 */
    unsigned int mask = 0700;
    static const char* perm[] = {"---","--x","-w-","-wx","r--","r-x","rw-","rwx"};
 
    char type;
 
    if(S_ISREG(st_mode))
        type = '-';
    else if(S_ISDIR(st_mode))
        type = 'd';
    else if(S_ISCHR(st_mode))
        type = 'c';
    else if(S_ISLNK(st_mode))
        type = 'l';
    else
        type = '?';
 
    permission[0] = type;
 
    /* 计算权限*/
    int i = 3;
    char* ptr = permission + 1;
    while(i>0)
    {
        *ptr++ = perm[(st_mode & mask)>>(i-1)*3][0];
        *ptr++ = perm[(st_mode & mask)>>(i-1)*3][1];
        *ptr++ = perm[(st_mode & mask)>>(i-1)*3][2];
        i--;
        mask>>=3;
    }
}
 
/*输出文件信息*/
void print_file_info(info *file_info)
{
    /*格式转换*/
    unsigned fsize = file_info->size;
    char ftime[64];
    strcpy(ftime,ctime(&file_info->mod_time));
    ftime[strlen(ftime) - 1] = '\0';
 
    int i = 8;
    printf("%10s ",file_info->permission);
    printf("%3i ",file_info->link_num);
    printf("%14s",uid_to_name(file_info->owner));
    printf("%14s ",gid_to_name(file_info->group));
    printf("%8u ",fsize);
    printf("%26s ",ftime);
    printf("%s \n",file_info->name);
 
}
 
/*遍历处理所有文件*/
void search_file_info()
{
 
    struct stat file_stat;
 
    int cur = 0;
    info file_info;
 
    /*遍历文件*/
    while(cur < size_of_path)
    {
        file_item* cur_node = files + cur;
        memset(file_info.permission,'\0',sizeof(file_info.permission));
        if(stat(cur_node->dir_name,&file_stat) == -1)
        {
            perror("Can't get the information of the file.\n");
            continue;
        }
 
        /*获取文件权限*/
        getPermission(file_stat.st_mode,file_info.permission);
 
        /*用户 ID 和 组ID*/
        file_info.owner = file_stat.st_uid;
        file_info.group = file_stat.st_gid;
        /*修改时间*/
        file_info.mod_time = file_stat.st_atime;
        /*文件大小*/
        file_info.size = file_stat.st_size;
        /* i-node 编号*/
        file_info.i_node = file_stat.st_ino;
        /*链接数*/
        file_info.link_num = file_stat.st_nlink;
        /*拷贝文件名*/
        strcpy(file_info.name,cur_node->d_name);
 
        /*打印文件信息*/
        print_file_info(&file_info);
        cur++;
    }
}
 
/*根据初始路径是文件还是目录区分处理*/
void init(char* pathname){
    size_of_path = 0;
    struct stat file_stat;
    if(stat(pathname,&file_stat) == -1)
    {
        perror("Can't get the information of the given path.\n");
        return;
    }
    /*普通文件*/
    if(S_ISREG(file_stat.st_mode))
    {
        size_of_path = 1;
        char* base_name = basename(pathname);
        strcpy(files[0].d_name ,base_name);
        strcpy(files[0].dir_name,pathname);
        return;
    }
    /*目录文件*/
    if(S_ISDIR(file_stat.st_mode))
    {
        /*统一目录格式*/
        if(pathname[strlen(pathname)-1] != '/')
        {
            char* ptr = pathname + strlen(pathname);
            *ptr++ = '/';
            *ptr = 0;
        }
        get_dir_detail(pathname);
        return;
    }
}
 
void print_simple()
{
    int max_len = 0;
    int num_in_row = 0;
    int num_in_col = 0;
 
    getWidth();
 
    /*获取最大文件名长度*/
    int i = 1;
    max_len = strlen(files[0].d_name);
    while(i < size_of_path)
    {
        int cur_len = strlen(files[i].d_name);
        max_len = max_len > cur_len?max_len:cur_len;
        i++;
    }
    max_len += 2;
 
    /*计算每行数目和每列数目*/
    num_in_row = terminalWidth/max_len;
    if(size_of_path % num_in_row == 0)
        num_in_col = size_of_path / num_in_row;
    else
        num_in_col = size_of_path / num_in_row + 1;
 
    i = 0;
    while(i < num_in_col)
    {
        int j;
        for(j = 0;j < num_in_row;++j)
        {
            file_item* cur = files + i + j*num_in_col;
            printf("%-*s",max_len,cur->d_name);
        }
        printf("\n");
        i++;
    }
}
 
int main(int argc,char* argv[])
{
 
    is_detail = 0;
 
    /*设置初始路径*/
    char initpath[256];
    if(argc == 1 || (argc == 2 && strcmp(argv[1],"-l") == 0))
    {
        strcpy(initpath,"./");
 
        /*获取目录下所有文件*/
        init(initpath);
 
        qsort(files,size_of_path,sizeof(files[0]),cmp);
 
        /*打印每个文件信息*/
        if(argc == 1)
            print_simple();
        else
            search_file_info();
        exit(0);
    }
 
    int i = 1;
    if(argc > 2 && strcmp(argv[1],"-l") == 0)
    {
        i++;
        is_detail = 1;
    }
 
    int flag = 1;
    while(i < argc)
    {
        if(!flag) printf("\n\n");
        flag = 0;
        strcpy(initpath,argv[i]);
 
        /*获取目录下所有文件*/
        init(initpath);
 
        if(size_of_path == 0)
        {
            perror("usage error\n");
            exit(-1);
        }
 
        qsort(files,size_of_path,sizeof(files[0]),cmp);
 
        if(is_detail)
        {
            /*打印每个文件信息*/
            search_file_info();
        }
        else
        {
            /*打印简略信息*/
            print_simple();
        }
 
        i++;
    }
 
    return 0;
}
```

### 效果
贴一张效果图：