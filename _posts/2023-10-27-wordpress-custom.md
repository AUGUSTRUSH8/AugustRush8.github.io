---
layout: post
title: 'Wordpress定制记'
tags: [blog]
---

# 前记
搭建一个基于ripro theme的内容管理站点，有些组件不支持或者页面模板不支持的展示需求就需要自己定制了，因为自己是个前端小白，因此有些地方改的并不是很优雅。

# 页面模板篇
## 子专题页面模板
背景说明：个人页面展示需求，想要一个“子专题”展示页面，一开始尝试创建一个新页面，父级模板选择专题，但找不到子专题内容填充定制参数配置项，因此需要定制一个子专题的页面展示模板。
找到我的页面模板路径：`/var/www/html/wp-content/themes/ripro/pages`
新建一个`zhuantisub.php`文件，直接从专题模板把代码复制过来，改改请求参数就好了，我这里的需要单独展示的子专题父category层级id是8，因此新增了一个`parent`等于8的参数请求项。
```
<?php
/**
 * Template name: 子专题
 * Description:   A zhuanti page
 */

get_header();

$cat_args=array(
    'orderby' => 'name',
    'order' => 'ASC',
    'parent' => '8'
);
$categories=get_categories($cat_args);
$i = 0;
?>

<div class="container">
        <main class="site-main">

            <article id="post-<?php the_ID(); ?>" <?php post_class( 'post zhuanti' ); ?>>
              <div class="col-12">
                <div class="row zhuanti">
                        <?php foreach($categories as $category) : ?>
                                <?php
                                        $i++;
                                        if ($i>30) { break; } // 最多显示30个
                                                        $args = array( 'cat' => $category->cat_ID, 'posts_per_page' => 1 );
                                                        $posts = get_posts( $args );
                                                        if ($posts) {
                                                                foreach( $posts as $post ) {
                                                                        setup_postdata( $post );
                                                                        $bg_image = get_the_post_thumbnail_url();
                                                                        break;
                                                                }
                                                        }
                                                ?>
                                <div class="col-12 col-sm-6 col-md-4 col-lg-3">
                                                        <div class="zhuanti-tile">
                                                                <div class="zhuanti-tile__wrap">
                                                                        <div class="background-img lazyload" data-bg="<?php echo esc_url( $bg_image ); ?>">
                                                                        </div>
                                                            <div class="zhuanti-tile__inner">
                                                                <div class="zhuanti-tile__text inverse-text">
                                                                <a class="zhuanti-tile__name cat-theme-bg" href="<?php echo get_category_link( $category->term_id );?>" title="<?php echo $category->name;?>"><?php echo $category->name;?></a>
                                                                <div class="zhuanti-tile__description"><?php echo $category->count;?>篇文章</div>
                                                            </div>
                                                        </div>
                                                            </div>
                                                        </div>
                                                </div>
                                        <?php endforeach; ?>
                                </div>

            </article>

        </main>
</div>

<?php get_footer(); ?>
```

## 文章页页面模板
背景说明：原来的文章页面右侧的各种搜索、最近文章等组件我从后台管理页去除以后发现文章页面一直居左，并没有像预想中的一样占据页面中间，于是想到肯定是父级容器样式的设置问题，于是直接去改了文章模板页的源码。
```
<?php
	$sidebar = cao_sidebar();
	$column_classes = cao_column_classes( $sidebar );
	get_header();
?>
<div class="container">
	<div class="breadcrumbs">
		<?php echo dimox_breadcrumbs(); ?>
	</div>
	<div class="justify-content-center">
		<div class="col-12 col-md-9" style="margin: 0 auto">
			<div class="content-area">
				<main class="site-main">
					<?php while ( have_posts() ) : the_post();
						get_template_part( 'parts/template-parts/content', 'single' );
					endwhile; ?>
				</main>
			</div>
		</div>
	</div>
</div>

<?php if (_cao('disable_related_posts','1') && _cao('related_posts_style','grid')=='fullgrid') {
	get_template_part( 'parts/related-posts' );
}?>

<?php get_footer(); ?>

```

# 页面插件篇
## 好物推荐页面
背景说明：希望展示好物推荐页面，样式形如：`https://veryjack.com/goods/`
实现方式：因为这个需求，了解到了WP 古腾堡编辑器的强大，但它还不够强大，用原生组件设计起来着实费力，因此安装了`Stackable`这个插件进行增强，拖拽起来丝滑得不是一般。

## 书影音页面
背景说明：希望展示豆瓣电影、书籍、音乐列表，并且展示效果要好。
实现方式：安装`WP-Douban`插件，注意是手动安装的方式，WP 插件市场搜不到，到作者的[Github仓库页面](https://github.com/bigfa/wp-douban)去下载zip包，然后到WP后台上传压缩包安装，使用方面，简单使用的话可以直接在页面粘贴豆瓣电影地址即可，插件会帮你渲染成卡片的样式，如果希望用更好看的方式展示，可以参考开发者给的[使用说明文档](https://fatesinger.com/101050)

## 相册页面
背景说明：以类似瀑布流的方式展示相册Gallery。
实现方式：安装`Meow Gallery`插件。在编辑页面点击左上角`添加媒体`，勾选图片然后再点击左上角的`创建相册`即可轻松创建。

## 目录索引
背景说明：给文章生成目录提高阅读体验
实现方式：安装`Rich Table of Contents`插件。注意，刚装上插件不进行设置的话页面内容可能会不可见，安装好以后先进行设置，参考[设置指导](https://boke112.com/post/7343.html)

# 网站Logo篇
## 网站左侧顶部Logo
1、先使用网站`https://looka.com/`生成
2、再使用网站`https://www.remove.bg/zh`去除背景

## 网站Favico
使用网站`https://favicon.io/`进行创作

# 附录
ripro主题常见主题代码目录
```
修改首页路径:ripro/index.php
修改首页轮番幻灯:ripro/parts/home-mode/slider.php
修改首页’最新文章’ 文字:ripro/inc/core-ajax.php
修改提示’请登录后下载…’等下载小工具提示:ripro/inc/go.php
修改用户权限访问相关权限文章/页面:ripro/inc/core-functions.php
修改 登录/注册 成功后等相关提示:ripro/inc/core-ajax.php
修改后台’商城管理’模块:ripro/inc/admin/init.php
修改后台商城管理页面:ripro/inc/admin/page/index.php
修改后台卡密生成/下载 提示:ripro/inc/admin/page/cdk.php
修改后台充值用户余额页面:ripro/inc/admin/page/charge.php
修改后台订单:ripro/inc/admin/page/order.php
修改后台资源订单:ripro/inc/admin/page/paylog.php
修改后台 用户申请提现:ripro/inc/admin/page/ref.php
修改后台设置首页+用户小工具默认+后台小工具名字默认+文章右面小工具文字···(自己去看):ripro/inc/codestar-framework/options/widgets.theme.php
修改···(基础设置):ripro/inc/codestar-framework/options/metabox.theme.php
修改用户高级信息提示例如封禁用户提示:ripro/inc/codestar-framework/options/profile.theme.php
修改隐藏内容提示等等···:ripro/inc/codestar-framework/options/shortcoder.theme.php
修改后台设置网站风格文字···:ripro/inc/codestar-framework/options/taxonomy.theme.php
修改第三方登录路径目录:ripro/inc/oauth
修改用户的后台目录:ripro/pages/user
修改专题:ripro/pages/zhuanti.php
修改存档页面:ripro/pages/archives.php
修改第三方支付目录:ripro/shop
修改搜索无结果路径:ripro/parts/template-parts/content-none.php
修改右上角用户模块:ripro/parts/navbar.php
修改首页底部模块:ripro/parts/home-mode/diy-footer.php
修改评论数量文字:ripro/parts/home-mode/filter-bar.php
修改登录/注册 模块:ripro/parts/home-mode/popup-signup.php
修改文章左上角提示位置’当前位置’:ripro/parts/home-mode/video-box.php
修改文章右上角提示「管理员登录后显示’编辑’」:ripro/parts/home-mode/single-header.php
修改文章下方’相关推荐’:ripro/parts/home-mode/related-posts.php
修改首页提示’加载更多’:ripro/parts/home-mode/pagination.php
修改文章提示’加载更多’:ripro/parts/home-mode/pagination.php
修改文章右边下载小工具:ripro/parts/home-mode/filter-bar.php
修改文章下方 ‘上一篇、下一篇’:ripro/parts/home-mode/entry-navigation.php
修改 ‘收藏文章’提示:ripro/parts/home-mode/entry-format.php
修改文章右下方 ‘分享模块’:ripro/parts/home-mode/author-box.php
修改搜索模块:ripro/parts/home-mode/search.php
修改用户VIP模块:ripro/parts/home-mode/vip.php
```