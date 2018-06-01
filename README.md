# WebView-TableViewDemo 
### 主流新闻app详情页中所使用的webview和tableview混合开发技巧
如果要实现一个底部带有相关推荐和评论的资讯详情页，很自然会想到WebView和TableView嵌套使用的方案。

这个方案是WebView作为TableView的TableHeaderView或者TableView的一个Cell，然后通过kvo获取webview的contentSize，再更新TableHeaderView的高度，这个方案逻辑上最简单，也最容易实现，而且滑动效果也比较好。

如果通过一次性的设置整个webview的高度，webview会对整个高度HTML进行渲染，在实际应用中发现资讯内容很长而且带有大量图片和GIf图片的时候，APP内存占用会暴增，有被系统杀掉的风险。

但是在单纯的使用WebView的时候内存占用不会那么大，WebView会根据自身视口的大小动态渲染HTML内容，不会一次性的渲染素有的HTML内容，此方法不在讨论内。

#### 主流app新闻详情实现方案
今日头条和网易新闻通过Reveal查看视图结构，整个详情页最外层是ScrollView，WebView和
TableView都是它的subView，subview的frame都是一个窗口大小。再禁用webview和tableview自身的滑动属性，通过scrollview的滑动去改变它们俩的contentOffsize。这样就可以在滑动的过程中慢慢加载webview中html，避免一次性加载整个html，造成内存暴增，可以使内存一直保存在一个平稳的状态。

![](https://github.com/leoAntu/LWComment/blob/master/2018-05-31%2015_53_13.gif)

本demo参照此结构仿写：

```
- (void)initView{
    
    [self.contentView addSubview:self.webView];
    [self.contentView addSubview:self.tableView];
    
    [self.view addSubview:self.containerScrollView];
    [self.containerScrollView addSubview:self.contentView];


    self.contentView.frame = CGRectMake(0, 0, kWidth, kHeight * 2);
    self.webView.frame = CGRectMake(0, 0, kWidth, kHeight);
    self.tableView.frame = CGRectMake(0, kHeight, kWidth, kHeight);
}

#pragma mark - Observers
- (void)addObservers{
    [self.webView addObserver:self forKeyPath:@"scrollView.contentSize" options:NSKeyValueObservingOptionNew context:nil];
    [self.tableView addObserver:self forKeyPath:@"contentSize" options:NSKeyValueObservingOptionNew context:nil];
}

- (void)removeObservers{
    [self.webView removeObserver:self forKeyPath:@"scrollView.contentSize"];
    [self.tableView removeObserver:self forKeyPath:@"contentSize"];
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context{
    if (object == _webView) {
        if ([keyPath isEqualToString:@"scrollView.contentSize"]) {
            [self updateContainerScrollViewContentSize:0 webViewContentHeight:0];
        }
    }else if(object == _tableView) {
        if ([keyPath isEqualToString:@"contentSize"]) {
            [self updateContainerScrollViewContentSize:0 webViewContentHeight:0];
        }
    }
}


- (void)updateContainerScrollViewContentSize:(NSInteger)flag webViewContentHeight:(CGFloat)inWebViewContentHeight {
    
    CGFloat webViewContentHeight = flag==1 ?inWebViewContentHeight :self.webView.scrollView.contentSize.height;
    CGFloat tableViewContentHeight = self.tableView.contentSize.height;
    
    if (webViewContentHeight == _lastWebViewContentHeight && tableViewContentHeight == _lastTableViewContentHeight) {
        return;
    }
    
    _lastWebViewContentHeight = webViewContentHeight;
    _lastTableViewContentHeight = tableViewContentHeight;
    
    self.containerScrollView.contentSize = CGSizeMake(kWidth, webViewContentHeight + tableViewContentHeight);
    
    CGFloat webViewHeight = (webViewContentHeight < kHeight) ? webViewContentHeight : kHeight;
    CGFloat tableViewHeight = tableViewContentHeight < kHeight ? tableViewContentHeight : kHeight;
    
    CGRect webViewRect = self.webView.frame;
    webViewRect.size.height = webViewHeight <= 0.1 ? kHeight :webViewHeight;
    
    CGRect contentRect = self.contentView.frame;
    contentRect.size.height = webViewHeight + tableViewHeight;
    
    CGRect tableViewRect = self.tableView.frame;
    tableViewRect.size.height = tableViewHeight;
    tableViewRect.origin.y = webViewRect.size.height;
    
}

#pragma mark - UIScrollViewDelegate
- (void)scrollViewDidScroll:(UIScrollView *)scrollView{
    if (_containerScrollView != scrollView) {
        return;
    }
    
    CGFloat offsetY = scrollView.contentOffset.y;
    
    CGFloat webViewHeight = self.webView.frame.size.height;
    CGFloat tableViewHeight = self.tableView.frame.size.height;
    
    CGFloat webViewContentHeight = self.webView.scrollView.contentSize.height;
    CGFloat tableViewContentHeight = self.tableView.contentSize.height;
    CGRect contentRect = self.contentView.frame;
    if (offsetY <= 0) {
        contentRect.origin.y = 0;
        self.contentView.frame = contentRect;
        self.webView.scrollView.contentOffset = CGPointZero;
        self.tableView.contentOffset = CGPointZero;
    }else if(offsetY < webViewContentHeight - webViewHeight){
        self.webView.scrollView.contentOffset = CGPointMake(0, offsetY);
        contentRect.origin.y = offsetY;
        self.contentView.frame = contentRect;
    }else if(offsetY < webViewContentHeight){
        self.tableView.contentOffset = CGPointZero;
        self.webView.scrollView.contentOffset = CGPointMake(0, webViewContentHeight - webViewHeight);
    }else if(offsetY < webViewContentHeight + tableViewContentHeight - tableViewHeight){
        contentRect.origin.y =  offsetY - webViewHeight;
        self.contentView.frame = contentRect;
        self.tableView.contentOffset = CGPointMake(0, offsetY - webViewContentHeight);
        self.webView.scrollView.contentOffset = CGPointMake(0, webViewContentHeight - webViewHeight);
    }else if(offsetY <= webViewContentHeight + tableViewContentHeight ){
        self.webView.scrollView.contentOffset = CGPointMake(0, webViewContentHeight - webViewHeight);
        self.tableView.contentOffset = CGPointMake(0, tableViewContentHeight - tableViewHeight);
        contentRect.origin.y = self.containerScrollView.contentSize.height - self.contentView.frame.size.height;
        self.contentView.frame = contentRect;
    }
}
```



