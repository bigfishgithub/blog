## asd
```php


<?php
namespace app\service;

use app\admin\model\article\ContentRule;
use app\common\model\Attachment;
use QL\QueryList;
use think\Cache;
use think\Db;
use think\Exception;

class ArticleDetailService
{
    private static $REGEX_RULES = [
        // 规则1: 删除src以/开头的<img>标签 (不区分大小写，.匹配换行)
        'img_rule' => '/<img\s+[^>]*?src\s*=\s*["\']\/[^"\']*["\'][^>]*>/is',

        // 规则2: 删除包含XSMB123.com的<p>标签 (不区分大小写，.匹配换行)
        'p_rule' => '/<p\b[^>]*>(?:(?!<\/p>).)*?XSMB123\.com.*?<\/p>/is',

        // 规则3: 删除特定div标签 (.匹配换行)
        'div_rule' => '/<div\s+class="bg-info-subtle\s+rounded\s+p-3\s+mb-3\s+fs-6"[^>]*>.*?<\/div>/s'
    ];

    public function cleanContent($content, $nav = null)
    {
        $content = preg_replace('/<a.*?>/', '', $content);
        $content = str_replace('</a>', '', $content);
        if($nav){
            $content = preg_replace('/<nav>.*<\/nav>/is', $nav, $content);
        }
        // 第一层：删除特定div
        $content = preg_replace(self::$REGEX_RULES['div_rule'], '', $content);
        // 第二层：删除包含XSMB123的段落
        $content = preg_replace(self::$REGEX_RULES['p_rule'], '', $content);
        // 第三层：删除本地图片
        $content = preg_replace(self::$REGEX_RULES['img_rule'], '', $content);

        return $content;

    }

    public function celanXsmbContent($content)
    {
        $content = preg_replace('/<a.*?>/', '', $content);
        $content = str_replace('</a>', '', $content);
        // 第一层：删除特定div
        $content = preg_replace(self::$REGEX_RULES['div_rule'], '', $content);
        // 第二层：删除包含XSMB123的段落
        $content = preg_replace(self::$REGEX_RULES['p_rule'], '', $content);
        // 第三层：删除本地图片
        $content = preg_replace(self::$REGEX_RULES['img_rule'], '', $content);

        return $content;
    }

    public function getSoiDetail($articleInfo)
    {
        $link = $articleInfo['origin_link'];
        $html = Db::name('sprider_html')
            ->field("id,html")->where('link', $link)->find();

        if (!empty($html)) {
            $html = $html['html'];
        } else {
            $html = CommonService::httpCurl($link);
            Db::name('sprider_html')->insert(['link' => $link, 'html' => $html]);
        }

        $ql = QueryList::html($html);
        $content = $ql->find('.content')->html();


        $nav = $ql->find('#ez-toc-container nav')->html();
        $nav = "<nav>" . $nav . "</nav>";
        $content = $this->cleanContent($content, $nav);
        $update = [
            'is_detail' => 1,
            'content' => $content,
        ];
        Db::name('sprider_article')
            ->where('id', $articleInfo['id'])
            ->update($update);
        return ['res' => 1, 'msg' => '更新文章详情成功'];
    }

    public function getXsmbDetail($articleInfo)
    {
        $link = $articleInfo['origin_link'];
        $html = Db::name('sprider_html')
            ->field("id,html")->where('link', $link)->find();

        $html = $this->http_curl($link);
        //Db::name('sprider_html')->insert(['link'=>$link,'html'=>$html]);

        $ql = QueryList::html($html);

        $content = $ql->find('.the__article')->html();

        $header = $ql->find('.list__subpage')->html();
        $content = str_replace($header, ' ', $content);
        $imgIcon = $ql->find('.align-items-center')->html();

        $content = str_replace($imgIcon, ' ', $content);

        $ads = $ql->find('.adsbygoogle')->htmls()->toArray();

        if ($ads) {
            foreach ($ads as $item) {
                if ($item) {
                    $content = str_replace($item, ' ', $content);
                }
            }
        }

        $content = $this->cleanContent($content);
        $update = [
            'is_detail' => 1,
            'content' => $content,
        ];

        //更换face
        /*
        $chnanelId = $articleInfo['channel_id'];
        $chnanelFace = config('temp.chanel_to_face');
        $cate = $chnanelFace[$chnanelId] ?? '';

        if($cate) {
            $info = Db::name('attachment')
                ->field('url')
                ->where(['category' => $cate])
                ->orderRaw('rand()')
                ->find();

            var_dump($info);exit;
        }
        */


        Db::name('sprider_article')
            ->where('id', $articleInfo['id'])
            ->update($update);
        return ['res' => 1, 'msg' => '更新文章详情成功'];
    }

    function http_curl($url)
    {

        // 1. 初始化
        $ch = curl_init();
        // 2. 设置选项，包括URL
        curl_setopt($ch, CURLOPT_URL, $url);
        //curl_setopt($ch,CURLOPT_HEADER,0);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);//测试号写上这个是跳过SSL证书检查，返回结果才不会null;
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);//禁止curl资源直接输出
        // 3.抓取url并把它传递给服务器
        $opt = curl_exec($ch);
        // 4. 释放curl句柄
        curl_close($ch);
        return $opt;
    }


}
```