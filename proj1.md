# proj1

- 主要是细节上的把控，大体框架注释里都给出了，按照instruction书写就可以。我遇到的值得记录的点有
  - 对于并发的控制，加的是大粒度的锁。使用c++ 17的std::scope_lock{latck_},在进入方法时上锁，退出scope的时候解锁，不会出现死锁问题。为了提高效率可以分析后加较细粒度的锁，但我没有做这个操作，太费脑子。
  - 由于DeallocatePage的时候没有写回磁盘，导致某些测试会出现ReadPage past end of file的IO错误，原因为新建一个page之后没有写回磁盘就读磁盘，导致磁盘内没有文件可以读。但检查代码后发现读盘操作没有发生在new page而是发生在fetch page，fetch的时候如果发现page_table中没有表项，就要选择一个牺牲页，从磁盘中读取数据再返回。而由于这个错误时显时不显，应为并发错误。情形应该是这样：
    1. 新建一个page，添加到page_table
    2. 某个线程将该页删掉，将表项从page_table中移除，但并没有写回磁盘
    3. 某个线程fetch该页，由于在page_table中找不到该页表项，于是尝试磁盘中读取数据，但因为页面内容没有写回磁盘，因此报IO错误。
    4. 当3和2调换顺序时，可以正常执行通过。
  - 由于unpinPage的时候没有检查page_table中是否存在页面而直接取值，导致在page_table不包含页面的时候返回true，从而一个case无法通过