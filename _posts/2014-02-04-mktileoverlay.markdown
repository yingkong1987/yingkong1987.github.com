---
layout: post
title: "MKTile​Overlay,MKMap​Snapshotter 和 MKDirections"
date: 2014-02-04 02:56:59 +0800
comments: true
categories: 
---

<!-- more -->

本文翻译自:[http://nshipster.com/mktileoverlay-mkmapsnapshotter-mkdirections/](http://nshipster.com/mktileoverlay-mkmapsnapshotter-mkdirections/)

除非你经常使用`MKMapView`,否则你可能听说过iOS地图制作的现状也许已经不是[非常乐观的环境](http://www.apple.com/letter-from-tim-cook-on-maps/).即使到现在,在那些所谓的专家已经转向iOS7独特的"外观和感觉",  `苹果地图` 这个品牌依然不能给予一般开发者信心.

因此,它可能在新iOS发布时惊奇的带来一个更好的地图.实际上很不错了---特别是iOS7引入的新地图API.这些新的API不仅显示了地图中先进的观念.但是规定MapKit工作区的局限性.

这一周的NSHipster,我们主要介绍`MKTileOverlay`,`MKMapSnapshotter`和`MKDirections`:三个iOS7中MapKit的新API,这将开启一个新的世界.



###MKTileOverlay

不喜欢苹果默认的瓦片地图吗?[MKTileOverlay ](https://developer.apple.com/library/ios/documentation/MapKit/Reference/MKTileOverlay_class/Reference/Reference.html)可以允许你无缝切换到另外的瓦片,只需要设置几行代码而已.就像[OpenStreetMap ](http://www.openstreetmap.org)和[Google Maps](https://maps.google.com),MKTileOverlay使用的是[spherical mercator projection (EPSG:3857).](http://en.wikipedia.org/wiki/Mercator_projection#The_spherical_model)

####设置自定义瓦片地图层

{% highlight C++ %}
static NSString * const template = @"http://tile.openstreetmap.org/{z}/{x}/{y}.png";

MKTileOverlay *overlay = [[MKTileOverlay alloc] initWithURLTemplate:template];
overlay.canReplaceMapContent = YES;

[self.mapView addOverlay:overlay
                   level:MKOverlayLevelAboveLabels];
{% endhighlight %}

MKTileOverlay用一个名template的URL字符串来初始化,X&Y表示瓦片在指定缩放级别上的坐标.[MapBox在这个方案中有一个关于生成瓦片的详细说明](https://www.mapbox.com/developers/guide/):

>每个瓦片有一个Z坐标来描述缩放等级,还有X,Y坐标描述他们在方格网的位置.
因此,最早的瓦片在地图网系统中是0/0/0.

![](/images/20140204/1.png)

>缩放等级0 是覆盖整个地球. 下一个缩放级别z0等分成了4个正矩形,分别是1/0/0 和1/1/0覆盖北半球,1/0/1和 1/1/1 覆盖南半球.
![](/images/20140204/2.png)

缩放等级彼此相关主要是4个力量 - z0包含一个瓦片,z1包含4个瓦片,z2包含16个,以此类推.因为每一个具体的指数关系数量以缩放等级增长.但是这样带宽数量和仓库也需要提供瓦片.

在设置`canReplaceMapContent`为`YES`后,这个覆盖层已经被添加到`MKMapView`上了.

在地图视图的delegate中,`mapView:rendererForOverlay:`是简单的实现在调用`MKTileOverlay`的overlay时返回一个新的`MKTileOverlayRenderer`实例.

{% highlight C++ %}
#pragma mark - MKMapViewDelegate

- (MKOverlayRenderer *)mapView:(MKMapView *)mapView
            rendererForOverlay:(id <MKOverlay>)overlay
{
    if ([overlay isKindOfClass:[MKTileOverlay class]]) {
        return [[MKTileOverlayRenderer alloc] initWithTileOverlay:overlay];
    }

    return nil;
}
{% endhighlight %}

####实现继承于MKTileOverlay的自定义行为

假如你需要在你的服务器上去适应不同的瓦片坐标方案,或者想要添加内存/离线缓存,这个可以通过子类化MKTileOverlay并覆写`URLForTilePath:`和`loadTileAtPath:result:`

{% highlight C++ %}
@interface XXTileOverlay : MKTileOverlay
@property NSCache *cache;
@property NSOperationQueue *operationQueue;
@end

@implementation XXTileOverlay

- (NSURL *)URLForTilePath:(MKTileOverlayPath)path {
    return [NSURL URLWithString:[NSString stringWithFormat:@"http://tile.example.com/%d/%d/%d", path.z, path.x, path.y]];
}

- (void)loadTileAtPath:(MKTileOverlayPath)path
                result:(void (^)(NSData *data, NSError *error))result
{
    if (!result) {
        return;
    }

    NSData *cachedData = [self.cache objectForKey:[self URLForTilePath:path]];
    if (cachedData) {
        result(cachedData, nil);
    } else {
        NSURLRequest *request = [NSURLRequest requestWithURL:[self URLForTilePath:path]];
        [NSURLConnection sendAsynchronousRequest:request queue:self.operationQueue completionHandler:^(NSURLResponse *response, NSData *data, NSError *connectionError) {
            result(data, connectionError);
        }];
    }
}

@end
{% endhighlight %}

###MKMapSnapshotter

另一个在iOS7中新加入的就是[MKMapSnapshotter](https://developer.apple.com/library/ios/documentation/MapKit/Reference/MKMapSnapshotter_class/Reference/Reference.html), 是地图视图的图像显示创建的正式过程.在以前,这将涉及UIGraphicsContext的,但是现在图像可以可靠的创建任何部位和透视图.

####创建一个Map View Snapshot

{% highlight C++ %}
MKMapSnapshotOptions *options = [[MKMapSnapshotOptions alloc] init];
options.region = self.mapView.region;
options.size = self.mapView.frame.size;
options.scale = [[UIScreen mainScreen] scale];

NSURL *fileURL = [NSURL fileURLWithPath:@"path/to/snapshot.png"];

MKMapSnapshotter *snapshotter = [[MKMapSnapshotter alloc] initWithOptions:options];
[snapshotter startWithCompletionHandler:^(MKMapSnapshot *snapshot, NSError *error) {
    if (error) {
        NSLog(@"[Error] %@", error);
        return;
    }

    UIImage *image = snapshot.image;
    NSData *data = UIImagePNGRepresentation(image);
    [data writeToURL:fileURL atomically:YES];
}];
{% endhighlight %}
首先,创建了一个`MKMapSnapshotOptions`对象,用来指定范围,尺寸和比例.
并用[照相机](https://developer.apple.com/library/mac/documentation/MapKit/Reference/MKMapCamera_class/Reference/Reference.html)来着色地图图像.

然后,这些选项传到一个新的`MKMapSnapshotter`实例,用`startWithCompletionHandler:`异步创建图像.在这个示例中,一个表示图像的PNG已经写入到磁盘.


####在Map View Snapshot中绘制Annotations

无论如何,他只在指定的范围内绘制;Annotations被分别描绘.
包含annotations,也许真的,任何地图快照的附加信息,可以向下拖动到Core Graphics:

{% highlight C++ %}
[snapshotter startWithQueue:dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)
              completionHandler:^(MKMapSnapshot *snapshot, NSError *error) {
      if (error) {
          NSLog(@"[Error] %@", error);
          return;
      }

      MKAnnotationView *pin = [[MKPinAnnotationView alloc] initWithAnnotation:nil reuseIdentifier:nil];

      UIImage *image = snapshot.image;
      UIGraphicsBeginImageContextWithOptions(image.size, YES, image.scale);
      {
          [image drawAtPoint:CGPointMake(0.0f, 0.0f)];

          CGRect rect = CGRectMake(0.0f, 0.0f, image.size.width, image.size.height);
          for (id <MKAnnotation> annotation in self.mapView.annotations) {
              CGPoint point = [snapshot pointForCoordinate:annotation.coordinate];
              if (CGRectContainsPoint(rect, point)) {
                  point.x = point.x + pin.centerOffset.x -
                                (pin.bounds.size.width / 2.0f);
                  point.y = point.y + pin.centerOffset.y -
                                (pin.bounds.size.height / 2.0f);
                  [pin.image drawAtPoint:point];
              }
          }

          UIImage *compositeImage = UIGraphicsGetImageFromCurrentImageContext();
          NSData *data = UIImagePNGRepresentation(compositeImage);
          [data writeToURL:fileURL atomically:YES];
      }
      UIGraphicsEndImageContext();
}];
{% endhighlight %}


###MKDirections

MapKit在iOS7最后的这个新增就是[MKDirections.](https://developer.apple.com/library/mac/documentation/MapKit/Reference/MKDirections_class/Reference/Reference.html)

顾名思义,MKDirections是获取两个位置坐标间的路径. 一个`MKDirectionsRequest`对象是用一个起点和终点来初始化.然后传递到一个`MKDirections`对象.可以计算几个可能的范围,并估算大概经过的时间.

`calculateDirectionsWithCompletionHandler:`是异步的,并返回一个`MKDirectionsResponse`对象或者一个`NSError`描述为什么请求失败.`MKDirectionsResponse`对象包含一个路径数组:`MKRoute`对象和`MKRouteStep`对象的数组. 一个在地图上多叉线的形状,并且有其他信息比如交通距离和一些有效的旅行公告.

编译前面的示例,这里是`MKDirections`可能被用来创建表示在两个点（可能随后将其粘贴到电子邮件或缓存在磁盘上）之间的路径计算每一步的图像的数组：

####获取方向上每一步的地图快照.

{% highlight C++ %}
NSMutableArray *mutableStepImages = [NSMutableArray array];

MKDirectionsRequest *request = [[MKDirectionsRequest alloc] init];
request.source = [MKMapItem mapItemForCurrentLocation];
request.destination = nil;//...;

MKDirections *directions = [[MKDirections alloc] initWithRequest:request];
[directions calculateDirectionsWithCompletionHandler:^(MKDirectionsResponse *response, NSError *error) {
    if (error) {
        NSLog(@"[Error] %@", error);
        return;
    }

    MKRoute *route = [response.routes firstObject];
    for (MKRouteStep *step in route.steps) {
        [snapshotter startWithCompletionHandler:^(MKMapSnapshot *snapshot, NSError *error) {
            if (error) {
                NSLog(@"[Error] %@", error);
                return;
            }

            UIImage *image = snapshot.image;
            UIGraphicsBeginImageContextWithOptions(image.size, YES, image.scale);
            {
                [image drawAtPoint:CGPointMake(0.0f, 0.0f)];

                CGContextRef c = UIGraphicsGetCurrentContext();
                MKPolylineRenderer *polylineRenderer = [[MKPolylineRenderer alloc] initWithPolyline:step.polyline];
                if (polylineRenderer.path) {
                    [polylineRenderer applyStrokePropertiesToContext:c atZoomScale:1.0f];
                    CGContextAddPath(c, polylineRenderer.path);
                    CGContextStrokePath(c);
                }

                CGRect rect = CGRectMake(0.0f, 0.0f, image.size.width, image.size.height);
                for (MKMapItem *mapItem in @[response.source, response.destination]) {
                    CGPoint point = [snapshot pointForCoordinate:mapItem.placemark.location.coordinate];
                    if (CGRectContainsPoint(rect, point)) {
                        MKPinAnnotationView *pin = [[MKPinAnnotationView alloc] initWithAnnotation:nil reuseIdentifier:nil];
                        pin.pinColor = [mapItem isEqual:response.source] ? MKPinAnnotationColorGreen : MKPinAnnotationColorRed;

                        point.x = point.x + pin.centerOffset.x -
                            (pin.bounds.size.width / 2.0f);
                        point.y = point.y + pin.centerOffset.y -
                            (pin.bounds.size.height / 2.0f);
                        [pin.image drawAtPoint:point];
                    }
                }

                UIImage *stepImage = UIGraphicsGetImageFromCurrentImageContext();
                [mutableStepImages addObject:stepImage];
            }
            UIGraphicsEndImageContext();
        }];
    }
}];
{% endhighlight %}