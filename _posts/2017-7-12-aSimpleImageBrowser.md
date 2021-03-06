---
layout: post
title: iOS - 简单的图片浏览器
category: original
tags: iOS,Objective-C
---
![picture]({{site.baseurl}}/assets/original/simpleImageBrowser.gif)
[GitHub](https://github.com/SilverJkm/JMImageBrowser)

<!-- more -->

{% highlight objc %}
 /**
 初始化方法
 @param urls 图片地址数组
 @param index 选中的位置
 @param rectBlock 返回动画需要的Rect
 @param scrollBlock 滚动到新Cell的回调
 @return JMImageBrowser
 */
- (instancetype)initWithUrls:(NSArray <NSString *>*)urls
     	   index:(NSUInteger)index
     			   rectBlock:(CGRect(^)(NSUInteger index))rectBlock
     			 scrollBlock:(nullable void(^)(NSUInteger index))scrollBlock;
     {% endhighlight objc %}

### 显示和隐藏动画的思路
1. 使用选中的Index初始化ViewController，并形成VC自身的循环引用
2. 将collectionView的offset设置到当前Index
  {% highlight objc %}
  	_cycleSelf = self;
  	_urls = urls;
  	_rectBlock = rectBlock;
  	_scrollBlock = scrollBlock;
  	self.currentIndex = index;
  	UICollectionViewFlowLayout *layout = (UICollectionViewFlowLayout *)_collectionView.collectionViewLayout;
  	_collectionView.contentOffset = CGPointMake(layout.itemSize.width * index, 0);
  {% endhighlight objc %}

3. 在VC的ViewWillAppear处，根据是否存在缓存，来决定是否执行动画
4. 动画执行时，应该是先用一个黑色的遮罩挡住之前显示的内容，再在其上添加一个ImageView专用来执行开始结束动画。

  * 对于开始动画，需要一个相对于window坐标系的Rect来作为动画的起始frame。
  * 对于结束动画，同样需要一个相对于window坐标系的Rect来作为动画的结束frame。
  * 使用rectBlock来返回相应的Rect
  * 坐标转换的方法 : `convertRect: toView:` ，将第二参数设置为nil，将默认转换到当前window的坐标系中



{% highlight objc %}
JMImageBrowser *vc = [[JMImageBrowser alloc]initWithUrls:originalArr index:selectIndex rectBlock:^CGRect(NSUInteger index) {
        //将index转为NSIndexPath
        NSIndexPath *path = [NSIndexPath indexPathForRow:index % kItemCountOfSecion inSection:index / kItemCountOfSecion];
        //获取在indexPath处的cell
        UICollectionViewCell *cell = [collectionView cellForItemAtIndexPath:path];
        CGRect cellWindowRect = [cell convertRect:cell.bounds toView:nil];
        return cellWindowRect;
    } scrollBlock:nil];
    [self.view.window addSubview:vc.view];
{% endhighlight objc %}

{% highlight objc %}
- (void)viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
    UIImage *CacheImage = [[SDImageCache sharedImageCache]imageFromCacheForKey:_urls[_currentIndex]];
    [self showAnimation:CacheImage];
    }
    {% endhighlight objc %}

{% highlight objc %}
- (void)showAnimation:(UIImage *)image{
    if (image == nil) {
        _coverView.hidden = YES;
        return;
    }

    CGRect fromViewWindowRect = _rectBlock(_currentIndex);
    _animationView.image = image;
    _animationView.frame = fromViewWindowRect;
    [UIView animateWithDuration:0.3 animations:^{
        _animationView.JM_Size = CGSizeMake([UIScreen mainScreen].bounds.size.width, [UIScreen mainScreen].bounds.size.width/image.size.width * image.size.height);
        _animationView.center = self.view.JM_BoundsCenter;
    } completion:^(BOOL finished) {
        _coverView.hidden = YES;
    }];
    }
- (void)dismissAniamtionWithImage:(UIImage *)image{
    if (image == nil) {
        [self.view removeFromSuperview];
        _cycleSelf = nil;
        return;
    }
    CGRect endWindowRect = _rectBlock(_currentIndex);
    _coverView.hidden = NO;
    _animationView.image = image;
    JMImageBrowserCell *cell = [_collectionView visibleCells].firstObject;
    _animationView.frame = [cell convertRect:cell.imageView.frame toView:nil];
    [UIView animateWithDuration:0.3 animations:^{
        _animationView.frame = endWindowRect;
    } completion:^(BOOL finished) {
        _coverView.hidden = YES;
        [self.view removeFromSuperview];
        _cycleSelf = nil;
    }];
    }
    {% endhighlight objc %}

### 图片加载与显示

> 先说一个问题吧:
     假设当前显示的cell，要显示的图片1本地还未存在缓存需要下载，并且此时由于快速滑动的原因，复用了这个Cell，去执行加载另一个图片2，那么之前图片1的操作仍然继续执行：更新进度条，下载结束显示图片1。
如果后加载的图片2先完成，需要显示图片2、去除进度信息，但是由于图片1的下载操作还在继续执行，那么最终的结果就会是显示图片1或者继续更新图片1的下载进度，而我们需要的是显示图片2、没有进度条。

 额，说了这么多，其实只需要给cell绑定一个当前需要显示或者更新的图片地址就可以解决问题了。



{% highlight objc %}
JMImageBrowserCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:JMImageBrowserCellIdentify forIndexPath:indexPath];
    cell.imageView.image = nil;
    [cell recoverSubviews];
    cell.delegate = self;
    NSURL *url = [NSURL URLWithString:_urls[indexPath.row]];
    cell.currentURL = url;
    __weak typeof(cell) weakCell = cell;
    [[SDWebImageManager sharedManager] loadImageWithURL:url options:0 progress:^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
        double progress = ((double)receivedSize) / ((double)expectedSize);
        dispatch_async(dispatch_get_main_queue(), ^{
            //判断该进度是否是当前cell要显示的图片的下载进度
            if (![weakCell.currentURL.absoluteString isEqualToString:targetURL.absoluteString]){
                return;
            }
            cell.progress = progress;
        });
    } completed:^(UIImage * _Nullable image, NSData * _Nullable data, NSError * _Nullable error, SDImageCacheType cacheType, BOOL finished, NSURL * _Nullable imageURL) {
        //判断此时是否是当前cell要显示的图片
        if (![weakCell.currentURL.absoluteString isEqualToString:imageURL.absoluteString] && cacheType == SDImageCacheTypeNone){
            return;
        }
    
        if ([imageURL.absoluteString hasSuffix:@".gif"]) {
            cell.imageView.animatedImage = [[FLAnimatedImage alloc]initWithAnimatedGIFData:data];
        }else{
            cell.imageView.image = image;
        }
        cell.progress = 0;
        cell.imageView.JM_Height = image.size.height * ( [UIScreen mainScreen].bounds.size.width / image.size.width);
        cell.imageView.JM_Width = [UIScreen mainScreen].bounds.size.width;
        cell.imageView.center = cell.contentView.JM_BoundsCenter;
    }];
    return cell;
{% endhighlight objc %}

### 关于进度条和百分比
使用自定义CALayer实现动画

{% highlight objc %}
- (void)setProgress:(double)progress{
    //当为0时，不会有动画
    if (progress != 0) {
        CABasicAnimation *progressAni = [CABasicAnimation animationWithKeyPath:@"progress"];
        if (progress >= 0.95 || _progress == 0) {
            progressAni.duration = 0.1; //95%以上时，加快动画速度
        }else{
            progressAni.duration = 5 * (progress - _progress);
        }
        progressAni.fromValue = @(_progress);
        progressAni.toValue = @(progress);
        progressAni.fillMode = kCAFillModeForwards;
        progressAni.removedOnCompletion = YES;
        [self.progressLayer addAnimation:progressAni forKey:@"progressAnimation"];
    }
    _progress = progress >= 1.0 ? 1.00 : progress;
    self.progressLayer.progress = _progress;
    }
    {% endhighlight objc %}

{% highlight objc %}
//进度条，百分比Layer
@interface JMImageBrowserProgrsssLayer : CALayer
@property(nonatomic,assign)CGFloat progress;
@end
{% endhighlight objc %}

{% highlight objc %}
@implementation JMImageBrowserProgrsssLayer
//判断当前指定的属性key改变是否需要重新绘制。
+ (BOOL)needsDisplayForKey:(NSString *)key{
    if ([key isEqualToString:@"progress"]) {
        return YES;
    }
    return [super needsDisplayForKey:key];
    }

//CoreAnimation动画时的帧，用于获取自定义变量progress
- (instancetype)initWithLayer:(JMImageBrowserProgrsssLayer *)layer{
    if (self = [super initWithLayer:layer]) {
        self.progress = layer.progress;
    }
    return self;
    }

//设置progress也直接进行重绘
- (void)setProgress:(CGFloat)progress{
    _progress = progress;
    [self setNeedsDisplay];
    }

//实现绘制
- (void)drawInContext:(CGContextRef)ctx{
    if (_progress == 0) {
        return;
    }
    CGContextSetStrokeColorWithColor(ctx, [[UIColor whiteColor] colorWithAlphaComponent:0.9].CGColor);
    CGFloat centerX  = self.frame.size.width * .5;
    CGPoint center = CGPointMake(centerX, centerX);
    CGFloat pathWidth = 5;
    CGFloat radius = centerX - pathWidth;
    UIBezierPath *path = [UIBezierPath bezierPath];
    path.lineWidth = pathWidth;

    //进度条
    CGFloat ratio = 0;
    if (_progress <= 0.25) {
        ratio = 2 * _progress + 1.5;
    }else{
        ratio = 2 * _progress - 0.5 - 0.0001;
    }
    [path addArcWithCenter:center radius:radius startAngle:M_PI * 1.5 endAngle:M_PI * ratio clockwise:YES];
    CGContextSetLineWidth(ctx, pathWidth);
    CGContextSetLineCap(ctx, kCGLineCapRound);
    CGContextAddPath(ctx, path.CGPath);
    CGContextStrokePath(ctx);

    //百分比
    UIFont *font = [UIFont systemFontOfSize:12];
    NSString *numberText = [NSString stringWithFormat:@"%.0f%%",_progress * 100];
    NSDictionary *attribute = @{NSFontAttributeName:font,
                                NSForegroundColorAttributeName:[UIColor whiteColor]};
    NSMutableAttributedString *string = [[NSMutableAttributedString alloc]initWithString:numberText attributes:attribute];
    CGContextSetTextMatrix(ctx, CGAffineTransformIdentity);
    CGContextTranslateCTM(ctx, 0, self.frame.size.height);
    CGContextScaleCTM(ctx, 1.0f, -1.0f);
    CGSize textSize = CGSizeMake( [numberText sizeWithAttributes:attribute].width, (NSUInteger)font.lineHeight + 1);
    CGFloat textY  = self.frame.size.height * 0.5 - textSize.height * 0.5;
    CGFloat textX = self.frame.size.width * 0.5 - textSize.width * 0.5;
    UIBezierPath *textPath = [UIBezierPath bezierPathWithRect:CGRectMake(textX, textY, textSize.width, textSize.height)];
    CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((__bridge CFAttributedStringRef)(string));
    CTFrameRef frame = CTFramesetterCreateFrame(framesetter, CFRangeMake(0, string.length), textPath.CGPath, NULL);
    CTFrameDraw(frame, ctx);
    CFRelease(framesetter);
    CFRelease(frame);
    }

{% endhighlight objc %}
