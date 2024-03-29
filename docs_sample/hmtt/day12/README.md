# 01-今日学习目标 01:07

- 能够熟悉docker搭建ElasticSearch的环境
- 能够掌握创建索引的思路
- 能够完成app端文章的搜索
- 能够完成app端搜索记录的管理
- 能够完成搜索关键词的联想功能 



# 02-docker安装es环境  07:04

# 03-docker安装kibana环境  04:07

  使用windows下的es进行安装，详见es课程中的windows安装说明



ELK ELASTIC-SEARCH(查询)  LOGSTASH（数据采集）  KIBANA（展示）



# 04-文章搜索 需求分析及创建索引 04:29

实现思路

- 需要把文章相关的数据存储到es索引库中
- 在搜索微服务中进行检索，查询的是es库，展示文章列表，需要根据关键字进行查询
- 在搜索的时候，用户输入了关键字，需要对当前用户记录搜索历史



```json
PUT app_info_article
{
    "mappings":{
        "properties":{
            "id":{
                "type":"text"
            },
            "publishTime":{
                "type":"date"
            },
            "layout":{
                "type":"short"
            },
            "images":{
                "type":"text"
            },
            "authorId": {
          		"type": "integer"
       		},
            "title":{
                "type":"text",
                "analyzer":"ik_smart"
            },
            "content":{
                "type":"text",
                "analyzer":"ik_smart"
            }
        }
    }
}

```

title.keyword字段是title的keyword版本



```java
package com.heima.search.config;

import lombok.Getter;
import lombok.Setter;
import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestHighLevelClient;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;

@Getter
@Setter
@Configuration
@ConfigurationProperties(prefix = "elasticsearch")
public class ElasticSearchConfig {
    private String host;
    private int port;

    @Bean
    public RestHighLevelClient client(){
        System.out.println(host);
        System.out.println(port);
        return new RestHighLevelClient(RestClient.builder(
                new HttpHost(
                        host,
                        port,
                        "http"
                )
        ));
    }
}
```



# 05-搜索微服务搭建  04:41

# 06-索引库数据导入 11:45

article集成es，编写测试类:

```java

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.heima.article.ArticleApplication;
import com.heima.article.service.ApArticleContentService;
import com.heima.article.service.ApArticleService;
import com.heima.model.article.pojos.ApArticle;
import com.heima.model.article.pojos.ApArticleContent;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.io.IOException;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@SpringBootTest(classes = ArticleApplication.class)
@RunWith(SpringRunner.class)
public class ArticleTest {

    @Autowired
    private ApArticleService articleService;

    @Autowired
    private RestHighLevelClient restHighLevelClient;

    @Autowired
    private ApArticleContentService apArticleContentService;

    @Test
    public void testImportAll() throws IOException {

        List<ApArticle> list = articleService.list();
        for (ApArticle apArticle : list) {
            Map<String, Object> map = new HashMap<String, Object>();
            map.put("id", apArticle.getId());
            map.put("publishTime", apArticle.getPublishTime());
            map.put("layout", apArticle.getLayout());
            map.put("images", apArticle.getImages());
            map.put("authorId", apArticle.getAuthorId());
            map.put("title", apArticle.getTitle());
            ApArticleContent one = apArticleContentService.getOne(Wrappers.<ApArticleContent>lambdaQuery().eq(ApArticleContent::getArticleId, apArticle.getId()));
            map.put("content", one.getContent());
            IndexRequest indexRequest = new IndexRequest("app_info_article").id(apArticle.getId().toString()).source(map);
            restHighLevelClient.index(indexRequest, RequestOptions.DEFAULT);
        }
    }
}
```



# 07-接口定义 06:20

```java
package com.heima.apis.search;

import com.heima.model.article.dtos.UserSearchDto;
import com.heima.model.common.dtos.ResponseResult;

public interface ArticleSearchControllerApi {

    /**
     *  搜索文章
     * @param userSearchDto
     * @return
     */
    public ResponseResult search(UserSearchDto userSearchDto);
}
```



```java
package com.heima.model.search.dtos;

import com.heima.model.common.annotation.IdEncrypt;
import com.heima.model.search.pojos.ApUserSearch;
import lombok.Data;

import java.util.Date;
import java.util.List;


@Data
public class UserSearchDto {

    // 设备ID
    @IdEncrypt
    Integer equipmentId;
    /**
    * 搜索关键字
    */
    String searchWords;
    /**
    * 当前页
    */
    int pageNum;
    /**
    * 分页条数
    */
    int pageSize;
    /**
    * 最小时间
    */
    Date minBehotTime;

    public int getFromIndex(){
        if(this.pageNum<1)return 0;
        if(this.pageSize<1) this.pageSize = 10;
        return this.pageSize * (pageNum-1);
    }

}
```



# 08-业务层 10：39

```java
package com.heima.search.service.impl;

import com.alibaba.fastjson.JSON;
import com.heima.model.common.dtos.ResponseResult;
import com.heima.model.common.enums.AppHttpCodeEnum;
import com.heima.model.search.dtos.UserSearchDto;
import com.heima.search.service.ApArticleSearchService;
import com.heima.search.service.ApUserSearchService;
import lombok.extern.log4j.Log4j2;
import org.apache.commons.lang3.StringUtils;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.index.query.*;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.sort.SortOrder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;


@Service
@Log4j2
public class ApArticleSearchServiceImpl implements ApArticleSearchService {

    @Autowired
    private RestHighLevelClient restHighLevelClient;

    @Autowired
    private ApUserSearchService apUserSearchService;

    @Override
    public ResponseResult esArticleSearch(UserSearchDto dto) throws IOException {
        //1.检查参数
        if (dto == null || StringUtils.isBlank(dto.getSearchWords())) {
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }

        //计算当前查询起始数据，是不是第一条
        if(dto.getFromIndex()==0){
            ResponseResult result = apUserSearchService.getEntryId(dto);
            if(result.getCode()!=AppHttpCodeEnum.SUCCESS.getCode()){
                return result;
            }
            apUserSearchService.insert((int)result.getData(),dto.getSearchWords());
        }

        //2.根据标题模糊分页查询
        //构建查询请求对象，指定查询的索引名称
        SearchRequest searchRequest = new SearchRequest("app_info_article");
        //创建查询条件构建器SearchSourceBuilder
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();

        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        //根据标题模糊查询
        //修改为match查询，模糊查询无法搜索 爱 最佳等字样
        MatchQueryBuilder title = QueryBuilders.matchQuery("title", dto.getSearchWords());

        boolQuery.must(title);
        //WildcardQueryBuilder wildcardQueryBuilder = QueryBuilders.wildcardQuery("title", dto.getSearchWords());
        //boolQuery.must(wildcardQueryBuilder);
        //小于当前的时间
        RangeQueryBuilder rangeQueryBuilder = QueryBuilders.rangeQuery("publishTime").lte(dto.getMinBehotTime());
        boolQuery.filter(rangeQueryBuilder);

        sourceBuilder.query(boolQuery);
        //按照发布时间倒序排序
        sourceBuilder.sort("publishTime", SortOrder.DESC);
        //分页查询
        //调用getFromIndex获取从第几条开始查
        sourceBuilder.from(dto.getFromIndex());
        sourceBuilder.size(dto.getPageSize());

        searchRequest.source(sourceBuilder);
        //查询,获取查询结果
        SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);

        //3.封装数据  获取Hits数据
        SearchHit[] hits = searchResponse.getHits().getHits();
        List<Map> articleList = new ArrayList<>();
        for (SearchHit searchHit : hits) {
            String sourceAsString = searchHit.getSourceAsString();
            //转为java对象
            Map goods = JSON.parseObject(sourceAsString, Map.class);
            articleList.add(goods);
        }
        return ResponseResult.okResult(articleList);
    }
}

```



# 09-测试 08:02

```json

#获取索引信息
GET app_info_article
#查看所有数据
GET app_info_article/_search
#获取分词结果
GET /app_info_article/_doc/1/_termvectors?fields=title 
#模糊搜索
GET app_info_article/_search
{
  "query":{
  "bool" : {
    "must" : [
      {
        "wildcard" : {
          "title" : {
            "wildcard" : "爱情",
            "boost" : 1.0
          }
        }
      }
    ],
    "filter" : [
      {
        "range" : {
          "publishTime" : {
            "from" : null,
            "to" : "2603-10-11T11:33:20.000Z",
            "include_lower" : true,
            "include_upper" : true,
            "boost" : 1.0
          }
        }
      }
    ],
    "adjust_pure_negative" : true,
    "boost" : 1.0
  }
}
}
```



# 10-文章自动审核构建索引 03:20

第一，在admin微服务中集成elasticSearch，参考搜索微服务

第二，在admin微服务中的自动审核代码中的saveAppArticle方法的最下方添加如下代码

```java
//TODO es索引创建
Map<String, Object> map = new HashMap<String, Object>();
map.put("id", apArticle.getId());
map.put("publishTime", apArticle.getPublishTime());
map.put("layout", apArticle.getLayout());
map.put("images", apArticle.getImages());
map.put("authorId", apArticle.getAuthorId());
map.put("title", apArticle.getTitle());
IndexRequest indexRequest = new IndexRequest("app_info_article").id(apArticle.getId().toString()).source(map);
try {
    restHighLevelClient.index(indexRequest, RequestOptions.DEFAULT);
} catch (IOException e) {
    e.printStackTrace();
}
```



>正常项目中，索引是不用在代码中创建的，项目上线前必须创建完索引
>
>logstash,maxwell



# 11-app端搜 录 05：58

- 展示用户的搜索记录5条
- 可以删除搜索记录



物理删除 : 从数据库中直接将数据delete

逻辑删除：使用status保存是否已被删除，0代表已被删除，1代表有效

select * from ap_user_search where status = 1 and entry_id = 1

表结构：

ap_user_search APP用户搜索信息表

![1587369069433](data:image/jpg;base64,iVBORw0KGgoAAAANSUhEUgAAAvoAAAB2CAYAAAC58jdfAAAgAElEQVR4nO3df3Bb9Z3o/feRXfLH9eyTYaDz3Js0CUaOZ/2omd52MMSGsoggkJ3lOnFqaEvXxXEstpqxBDTYdA2bC7mtTRYiaa66yDFu3KVt8I0T3za2wE3NUtZOCUOfZ71+3HUk1JAmO3Onz8Nmd8x/z4yeP86RdPTTsiRbP/x5zWiS8+t7Puf4nKPv93u+36+U7u7uMEIIIYQQQoiKUn3vvfcWOwYhhBBCCCFEgSk3b96M1uhPTk4C0NbWVrSAROWR62pjyHnOT6HPXyX8PSrhGNZDqV8rpZ6eEGLjVC8v/QaA+oZ74hYsL/2G/+Nz7xcjJlFB/r+6o3HTketNFFaq+1dkr9DnrxL+HpVwDOuh1K+VUk9PCLGxqhXFkHZh2HDLBoYihBBCCCGEKJRqxZA+o49hy8ZFIoQQQgghhCiYakOmGv2qj3HsOcWPEhd8qY1///FX1f9f+zX79v8vnlv4GpYsd/rxmx7+8ytXgb2czbDdzEvPcOhs5nUAfvgXz9D/f+1i8EIv3+HX7Ns/yeXEOEXpuOTm7mf9uhkNHHj1Nfr3Fi2iypRwng+8+rac47Uo+HX6IYNNL3BeN+eLbS8z8txdXH/rGdrdS0Dk7xRb94uONxh5bFuuOy2oMz2PcHKxgafHX+NxztPd4eOfAEw2Phg+UFL7i51TK655B3uzXJZ7nKjpje/gjXzPy3XdsaId//b8Yrz0yiM4J3Uz2l7mg+fuyj3B6x8y+NILnF8ETFZcD8IbHCqZa1UIoTIohioUQ1WapV/CPX2Ixq99h39f9Gqf7/Ak1Wptv2EL7HqIi4tPYIlMZ/G58y+O8u+LXga/VJVxPcux1dfBsIXvvPkdnvySwrmxoBaPl8Ev3cvZNx/KOib5rNMnlb0OJhwNHHj1bT6Yf5sP5p+FHz3DmetZXLHXz9P9yodrv9Jz3a6c7XXwwfzLHDDZ+GBeMvlrls91mtJd9I/b+GLby1p6b3N4588YvATbH3sNV1sDT49H/k53cX9bAwdejWTyb3Cm5xHubnqEu5vyiSE/jw+/zAETXPzph7D9ACPzb/O0yYorm8xsDvdgPvvb/thrfDD/Nk+b1rYsl3Otxmnjg3kHe9d6XlIGr6Yx4WjggGMnF+du5JZOHPX6+kBL94s7/1Mead1g8KV57n9RuzeGHcAnBYhxHWzGZ78QOgaDYiBdrX64agvhqmpQPke46gq933qfcNWXcf3sEcJVW5h5yc6fmOz8iektZqq2aOurn49/clJbZudPvvUWMzfil4erthBWDJC43fxb7NO22/fSlZTrJH8+R1i5m+eUGbw3ktOOT/N9Pq7aoh6Pyc6fmE6yT9uX91vqdCSNrI5BPhk/2dlG/5OxL7Prl9x0N6lftN2vnCf6PXvJzd0dPv5p8gXtS/gR7tY9wHPa7pJbne45nzR9pkf999Irz0S/9Acvxb5wr7/1TCy9HjeXipT5ylW6+NX5z9Dd80zRM5alJeE6LcDff++OnQSvJWbibnDmlWd4777X6N+r1o5eeuVVrj6pZdJe3cnJl84nJ7Zh7uMwP0t9TVw/z2BP7JxE11nl3s1lf5deeYS7m9xciqSvv4/zUOhznfv9dIN/+NVOnnjsEMZfXY4+zzIe9yV32tT2Pqe9Fbh+nmO/uo9jupr39Ndymr/RpbMEHzzEXt1bhr2PvRarzU95HXzIoHb83U2P0P3Kh1qB6hnOXM+0LHOMGc9vXtedEJXBoBgMpG2nb7gFlCoYd/O//amb01Sp87TPQy//mH9b/jE/+M/x8/nDu7gVG/+2rC7/tzN/TmD0XUL6dQy3AMnb2f72C/i07XzGBZ7/PxPWSfn5HFDFQ913c370n5PSrt3157o0/xe2v/tXMHwF98w3aeQLPDdzggMBN+etJ/i3U1/g/Hv/mv0xyCfzJ1tf2AGf/AsA279wiGNajeexndc49paWEdrr4IOEGlH9q+ecttvrUGtSXzyQNP348MscWPTx3s6va9u9xv3vn1W/RK6f502ejaU3fIhPfqorXJS6DPGrtctLYPx6LLPzU/lyBGLXaYH+/pfe/wTjDn1Thxuc6XkVvhHfRGjvc7Hp7XubKHwjmbXZ+4371Fr2OB8y+BI8MRw7J7ykZUhXuXdz2d/e53Q18pG3V/kcVDTdwp7rnO+n65e5aGxiO9u43/g+/3A9El+G497rWDXZMy+9z74XDxDNo2e8llP/ja5fS7xu9dJdB9obLXZyePwN9gVf4OKDb/DBqzu5OPefMiy7kfvzKs/rTohKsMqoO1vAUA2PP8/Nl8HZ8S/qvCRVhA1biAzIH3rvQ05//yec/r5+nd1w5FH+cmfm7RrsL3OHAcLAHU+28e3v/yJundQ+B1QTvqOFo8oL/PAPd8WlHXrvHZ76bxfVdvtA41+16Y7tHvbdsYMA+zj65A7C71WBcgsfZ30MotCuz53lmNuvtU+FLzoOret2e79xH2/89EMef+4uuH6eN/g6I5FvQZONJx6LfTHsvQ/emLvBvbzPebeP83EVaA3wjQN5t6XdCNfnVou/gX3fUI97+94mDvzoOum+9Dej1c9fBpMvcLfWVvqLjjcYiWbolzj50llcD+7kjZfOc++wLjMW3fGHDL70M3jytQIcRR62H+Awz3Dmuu6auDRP8MFDupi3ce+Dn/DmJdibb7OxVPtbbwU912u/n67PvY/xPnXfe+/byRtzN3g8z/bv1996hosPPht7vpHntZxKpuvgC0BbE3u3b+MTrBx+bBtqSVCTZpk8r4TIXbUhXft8gKotoDXdoerLuCbSrKc1k4movXMXjS88zcyT/zHz3hO2w6Co7bqjId0CJKyT0udAqYaqLTxku48Tw/83ByJpf/ILnvpvBo6+O8lDO4G//yGW39+ScGxbYrEYDOpbgDuyPAZRGH+4Bjsb1dfKbjg8/rb6WviSm+5rWWyf63YA2w+wL6hmIu7Vfblm3GTHTr7oeLb8Op5dctN97RAj5Rp/sWnXaV5//7SdIBt4+kUHe7fDF3iGNy8dSOpXcealee4ffi3vzqOFECkg76vQ/eV3rvNtU3+Df/jVEucXH4l13jbtgMfyeL8QabIzHH/N5nItb480Odu7Mc+Psn3eClECVm+6Y6hSm+9kbJ6RsNzcTMPUz/nlH1Zr1hG/Xe2++1n679PR5jGh0z9Pai6U+lMVS+uOdo4q45yPblcF39jFQ3fcAn/4f/nb/z4Dim67yLFF1lfUGv2sj0E+BWi6c4MzP/qEfc3aQ7xth5pZv36DMz/yJ68evK6+Vr7+IYM9jzAYqRHKdTvg8Sd3cvGn53nzV/fxRNw3+/u8+VbsNful97U49zZh/NXZsmuXH1Xu8ReF7jpd5/O3vfk+gj9Kbhv++HD+I8QUzPYDHOZnXIxM722Ka0seaWN+vz7gDPfgmvcHwCd8ch31nu+JH9EoX2s713dxv/H9aB+e65fOctHYlMX2aZrwXL/MRaOuucn827iM7+va9q/1uG8kNdk50xNpVpXDtbz3EPt+9aounhtceuUZ9e+ZzXWwVvneb/lcd0KUuVU64/4jzvuHufzTv2brrha2/tU/6jpa/iPOXS1s3dXC87/187VdLWzddZQf/mEL4ap7Oem+g6ln1OVbdx3lob+6QGi17Wof43XnDWzactvHX+X7X/HztfYLGTt8/rD9rzn92+FoOvv+0kz4twZ1ee1jfFeZU+O4/zXCj7Zy+aXDOH8dO7aHTv8/hJXIfgza8kzHIJ+8OuNectPuXuL8s5GRLV7l6pNaR7HtBzjM++r8jlfhQSv/5D4cezBvP8Bh4/u0Nz3C3R0/gwffUGs9c90uYq+DfUEfwQcbE5pL3McT/Czayeu9+57VXhXfRf+LO3jvpdjoHHEdgEvBJTd3N73A+UVfrCNadLjI9PFff+sZnJNLnOxwcwktI7Ho23yd2DJdpxn//unO04cM6jsG6juSR8+5lgnZfoB9+Li7J3ZNXdc6Xna/VYgRWHITvRa0Do97v3EfLEaW3kX/i/BmtBPmWXhRl1le7R5c8/60AnqHes9ffdDGgeh1Gunc+QgnF/044zppZlqmyuVc733u6/Cjw9zd9AjtP4LD2lubXO6n7sh1oq2jDo25xMkONc70x53Ov3B1Ub2+7o4ee2RZhmv5eroixDYeH34WfvpM9N54b+ez0RGjUl8HsetfPa9+nFon4n9yH452mk21bPBSHs+rHK47ISqJ8s+/+8cwwP/+H3cyOak2HG1ra2N56Tfs3llT1OBE+VP+gynpuipNNzjTc5Z7hx26jP6HDPZcp38dxgcvtPqGe8rkPJemQp+/Svh7VMIxrIdSv1ZKPT0hxMZa5Qez1jBqihApKMUOYFU3ONNzOFq7dfKVpmj7abVmCM43+eTHpoQQQghRdqrTts8H0v7gkRAVYxuPD7/N4ymWpJsvhBBCCFEOqm//fOpe7PUN92xwKGIzkOtqY8h5zk+hz18l/D0q4RjWQ6lfK6WenhBifSmnT5+ODlH/7W9/u5ixiAp2+vTpYocghBBCCLGpVBc7ALE5dHZ2FjuEijc1NUVra2uxwyhbhT5/lfD3qIRjWA+lfq2UenpCiI1T3dbWljTz4+AfixCKqER3Gm+P/v8P//KvRYyk9M3+8uc5b2t+6NHo/+U856fQ5y+f9F766+dy3vbF//pKztsm2ohrqlSOdS2OHDmS87apYi719OTZIkT5qa4OnCh2DEIITaqC92oiQ98JIcRGe7fvaRg6yQNZrHv11MP47nyHH5hTL3ua1zl/5I6CxyjEZlYdzvrXS4UQQhTDqVOnsl43n1rcUlCOx1romEs9Pb0Hhlp5ftutzPzdpykz8DG/ZzZ4AFs0+Yu8eyrI7+80gusExz76EPgKB4Lfp/cvn+KBXWsKQwiRhgHDFqKfRJ/4+JrxWf4+QwJ/P3A7d66yjhBiY12dvcho38Ps2PY0765hmShPilL6v1hRKOV4rIWOubTOwT5+MPd9/vmdi3Fz3z31Olf1M66+wy/e/B5f3XYrO7bdyo5tU8wEl7nDvA+z53V+fayTY3OfctK4zO83NH4hKlt1xh/F2mnjfwQzJ/Bnx//IC1eeLWxUorJdfZ0Dzcv03sjuda9Yu13mfXSZ98Hvnl7TMlF+SivTt77K8VgrLpM/+zQ7vjWWNPubT3SyY1tHbPrY9+HU7/mB1hTn3V/CyRufEq2ov/o6z//yYei7lad/dxcfffQhHBvjGMCxMQKrviEQQmSjOt2PYv39wO0cPgPwF7wRfJU/0y375L1n+e7hH/Nb4MuPn0H64os12fUU52+scZurr3Pgb42cH9q3LiEJUQ4URSEcDsdNV6pyPNZCx1yS58B8kms3TsbNUtven+Ta0MnU28w+zQz1/PO2W/kI4Invc+x33+MnLFP3p1omP+oujs29Q5c03RGiIAzhqi1EPnp/dvyPfBz8Iy98OWGLT3x817ubvwmqy//GOM3Lv924gEV5e7cv8to2vtnI1VMPs2Pbwxx49GFt+cOMRt77zj7Njubv8dGbHdqyW9nRdzFV8kJUvEhmT5/p02cGK0k5HmuhYy7Hc5DEfJIfHHmK3ifu4tjcp1wbeoqun3/KNU89v+Ao1258Gm26c+2GZPKFKKRq1tgZ95N3/yf19ml2atM7v23n68e9hY9MVKQHhj7l2hCMPhrfbGTXkXcYC96Kh3Gu/XwfV2ef5qt/e5GuoX1qDdJc/aas0d+6dWvSvJs3bxYhElFKyjrTt0bleKyFjrnUzsG7fbfS+Wbi3Fv5iX7yiXGuRZ7X+uY+b97KMeCbf/cpP2AZWObAtg61pj/SdEe/rRAiL4awYQuRjxDFdRd//pfqw32XuZVv/m6VDiKbQGKmXjL5m1tiJq8UMn3rpRyPtdAxl+o5eGDoU67diH1+fewuvvl38fPiMurmk1y78SljkRr9G59i4SJXP17iozfH1Ey+3ptTMlCAEAVioGoL0U8Wdj7wX1j2+vhEm/7ktJefrV98Qmx6kcy9ZPIFxDJ7pZLpW0/leKyFjrkcz0FmF3m+7yIPMMVXf3GAX0cLC53RQsA1GahBiIIxYLiF6CfqVwwYb+dO4+28/Nsfc9h4O3caWzj9CbDTxt/Yr/Bdbfl3gy288OUfc7jDV6xjEJvF74LqcG1XL/L8o7fy/GyxA9o4a83kjz6q9mU49tEYndtuZcejsaHuMi0T5aFyMn2rK8djLXTM5XgO4PdcTXqwfMix5iksQ/D8t+DYn5/nq/L8EWJdVSd2wlU9yPHgHzmeZqOd97/K/wi+qpvzR769DsGJSnOR57d1xNpxbhsjMsKC+ZcP0/nmh/Dm09xxo5ffP9rBTz6Cn/QZ1VfAu56i908f5qvbvgfcxTePfSRDr2XQ9fNP6cphmRBCpHX1dQ40fy+pqc1Xjn1E7zu3suNb+rl38ZWvNND781jt/ANDnzLGrXRugzGt1r7rodc5EBmNJ9JGX0beEaJg0g6vKUTh7eMHNz7lB6kWHXmHa7ofZHwgRWb0gaF3uDa0juEJIYRIb9dTnL/xVJqFn2b1fI4MyJBdmkKIfGX+wSwhhBBFd+TIkdVXqhDleKyFjrnU0xNClA+p0ReihExOThY7BCGEEEJUiGrDLTuTZt5pvL0IoQixuZkferTYIYgS8+J/faXYIWyYcjzWQsdc6ukJIcqPcvr06Wh3/m9/W7rUivVx4cKFYocghBBCCLGpVOsnTp8+TWdnZ7FiERVqamqK1tbWYodR8eQ856fQ568S/h6VcAzrodSvlVJPTwixcarb2tqiE5H2wZ999lmx4hEVTK6rzM6ePZvztocOHYr+X85zfgp9/vJJz+l05ryty+XKedtEG3FNlcqxrkU+nVxTxVzq6cmzRYjyU7289BsA6hvuKXIoQgh9wTtb0oFXCLGRQqEQtbW1ea8z09sLHg+WQgYnhIhTrSiGYscghBAig1OnTmW9brkPpViOx1romEs9Pfzd1PRdXmWlRoYWZrFnyOtbPPvprTETWGU9IUTuDIrBgGIo88x+yIu5ppeZDKvM9NZQs8o6QlSK0MwM3l5zyms+0zIhROEpilLsEArMxNDCCisr2udcF41DC7HplQWGutqxpsi8q9/Fkc9BRrlM3x79vPjnktesX1ZDjdkbSSh+fo2ZXnmgCZHEYFAMGApZqx/yYt7ou63WzuxK5td/Fs8KQ40bFpHIJIuCmchPrcWC3TOb8prPtEwIUViVl8kHWIzPnB8c5XLfHl2mew99i6m3NNZ3xRcS9J+FIRq79sd9l9tn1WVDjV2cW1lhZdauLrB4WBhqpOtcZPsRGDTjDa37wQtRVgyKoQrFUJW0IOQ160rQvcyE9PPNmM3maCk6emPN9FKzp4/Lowdj20Yz/TP0Rtaf8dJr1pXcIyXzxJJ6ZDqDWO1AqprLXsxaHBte+BDpZVEwS1KMAqQQQqxBYqa+MjP5kFWNvmmjY6rF029iwi85fSH0Utfoh7y4GIndtLNOAi4vIaDWPsu5rstg6mdlZYWFcyb6XFoGzOLRSuTnYtt6Itk5C56VFc51XaZvcJn6/gVWzsEFbwgsHs51NTI0Eiupx01nYPFESvsJC0JeugfrGdHiGKm/wKpNCsW6S1cwy70AKYQQpSOSuddn8sPhcLrVN53g8mhCUx3dZ08feX1NG+thOVioUIWoCCnb6If8E4wmvoYbnSBWUG6k3alm4Gst++laDKxhl40MjXiwW2rB4sGj9cCxONuZiBQYQl4G6c+rc07IP4Gp304kiVq7k67ckxMFkq5glnsBsrJt3bo16SOEKG2VnslPyqyvoemO2kk3TdOdTfRsF2KjGBTFQOLIO7V1poTXcCusrKxzr/haO+2Lg3hDWiZ9v9zsm08+BcjKdPPmzYzTQojSkZipr8RMPoQILGrt5dfcdGeGC4upO+kWRHAZ6o3rlLgQ5clgMFRhSGyjb9mPacIVbZe/ZosBQgChGXrNNVn3hLf3m5hweXFNtOPMM59fa21ncVBtbgQQ8roYzS9JIYoikrmXTL4QpS+Sua/MTD5AkGXT/tzGvp+5ALo37YUVwju4SPu6lSKEKE9phte04Bmp50J3bNgqc6+aaQ55zRwcvUzfnl5mCOE1H2T0cl+szXStnX7TBHtqaqjZMwjtC3gsaCOt1Gjbah1kE7vHWzy0L/ax2G7N8kEQ6eBbQ9/lUQ7q23bX2hnpX6ZbW969vJ+hxlEOZtHBV5SoHAuQlWCtmfzIkHTR+8IcK/RmWiaEyF/lZvIh5L1AfdqauBChdA+TkBfzhf2stWVOqucVADO97Om7zOjBSD6lm+V+GY9fiETVaYfWrLXjmbXjSZxvn2VF10fWMrtCYpdZi2eWlcQNa+3MrqzWuTYEdDGS9Z2qdvBNijGyS4uH2bhAkmMVG2mG3pqDsTcrNaNEflTF6lcLkIz2UrfiJGA+iDpZp7bZrLXTbzKzp6YPaKRraGHNXxibiT3FfZnNMiGESCvkxYUTT4qvaFOdOjPoqmHPKNB1LvacCXkxd8PI7Nof2mmfVxYPK0kZDSFEoupUQ2tuvBBe857oqDh9vfulQ05FylAwy7UAKYQQYmPU2vGkynVbPNHnusWzkrqib3adYxNCpJS+Rn9D1aYutYe8mDMMt9V1bkVqdYUQFe/IkSPFDmHDlOOxFjrmUk9PCFE+qpPb55eQrJr7CFE5Jicnix2CEEIIISpE9e2f31bsGIQQwKFDh4odgigxLper2CFsmHI81kLHXOrpCSHKj3L69Om44QFuu+22YsUihBBCCCGEKJDqxBmtra3FiENUsKmpKbmuNoCc5/wU+vxVwt+jEo5hPZT6tVLq6QkhNk51W1tbdELaBwtRPGNjYzlv29nZWcBIRKnIpxPlqVOnChjJ+ivHYy10zKWenhCi/FQvL/0GgPqGe4ocihBCX/DOlhTQhRAby49NOU5DYA6HUT/bhjLZRthnzTIZG8rxBgJzDoyrry2EyEG1UhLDawohhEhnLbWr5T6UYjkea6FjLvX0AGjqoNUIBIP4pzppcYLLZaKn4QrNSgvzTU30dIzhi5YEgrib63DOJydVpzh1Uz1Mh33EigradqZpwj4rQXczdakSAeiZzr6QIcQmYVAMBpKH2PRjUxQUm78oQeXKb1NQFBvlFbUQ6yDox9asoCgKSrMNfzB+sd/WTLOioDS7CaZOQQgh4gXd6nNDaWF43kmdomCbmmKSMcLhORy7gd0O5sJhwgMmFuM2NuKYCxMO6z7TPfRMJ8yLy+THtgscrVOnHHMJ6+vSapNMvhCJDAbFQPKPZlnxBVw0FSWk3Fl9YVzlFrRYN5u54OfuPE7DQIBwOExgAFo63dFlQbeNybY59ct4DE64JasvhMiCUcvEh6fpaXIRCIfxtcKis073rNUqCluG47f129SKB/2nZZjhFiV5fooKCKPRGEs78VNmlZJCbCSDYqhCMVRlXEnNMKk3H0DQ3ay7IXW1hZEbWVsvNt2s3ZzNRPMU0ZoB7eEQdMfVQMbyHv7Ytv7IOrHMW9Bv09JRaJabXehs5oKfY24Oh1V9ZW60ttGjWzY1DtGKL+NuGJ/a8PiEEBUg6MY9BSZXQFcTb8Wn1bDH+LG1LOIKRGrgp+mBWG3+dA9NrgDh8DQuV4BwpM1+NJ8QycxraSd+pLmOEGmlqdGPV9fQo96gcw4IujnBWOwGmzvKlRNa6dvqY7qnCdeYQ90wOj2Hb7qHJtdYrOOO0cHcdA9NrqNY8WPrhKNzsTTpjGTm1Rt7umce5/EltZZyGibdQQi66TzewJgWy1jDZMr2f6I0uJvjC4zRAqRWcIsvtOlrdDIX9uKaqSjN2Gz6giJM2pqjy/Tzc95fmQm6j7PYkW5oPCsNLJXtsQkhiiB4hcV5J3Wd0Bqt0U+ooY+r0bfiC6sdd9XnfgvDTS6OWrVn7WQbY63qeg59797IG4TpHpoa1KY7SW8GpPmhEBmlaaMfcQW3zcZUqy+aQQ9OjTMcuakVBUWpwzk8zpR2p1mPdjB+Qss2BN0cZ0Ddtq4BlgLaTdqMzQ/+yUU6Wo3gn2Sxo1XX695Ia8cik3G5jyZcYz61ltLqw+cwEpwaxzQQ661vdByNq7kUpcUxF8DV1MP0nFoQtPrU6YBWE2SsO6ortC3RGc2VZyjsEYxrphIOj9GwqG8ZOswwAwTCYQLTJpwnYhdVbvvbWFu3bk36rEXQbaNzaYA5h4xpIYTIUyST3TkOTS51tJxoc57kNvepnjtWn7ps2uSkTpmkLRwm3DZJXd2JrCsc1Np/rQAQl3cQQiQyKIqBtCPvDI8zvgi79QXs3abYTRb9zMXV1HcsHscdVAsFpkgbAWMrpsUrBK8s0uPqYHHSz5VFk9prX2wSRhwDcDySYfafYLzjaPQhHZw6QadWgKxzDqfYPrmwR3CKcdNAtJmK2nFLP+RbD9M+K0a0JiyLV6Kp5bS/DXbz5s2M05n4bc2c4ChzGV9rB1miAXnxLYRYldWnvXUfwKTNir2ZVWvm2yYz1LQHgwS1woJa4T9MS7T2X/1/c8YKlSDBKzCve4MghMjMYDBUYUjXRr9ngLmxBo7rb1hrG6bxE0mjeOg5BkyMn3BzYryDo9EchJE20xKd49DQ2krH4nHGTW1qJs/ahml8SvdQCDI1bmK1DvTG1g4Wj8diC7pPILd9ibMexTR+giBB3MdhIPqqyE2nEwYCsZqhdbXR+8tDJHOffSY/iLu5mcm2uWjhxN1si94nrR1wJdrJZSpDsx4hhMgsUkMfcDXR02bF2tajDnOZamx8Y4ATk22xSsKAix5XgPC0K9p+P/3bxylsygkCu+Nr9E27pbZQiEzSD69Z52QeILDEfGQYLT+AFd9YA5OdsTbR8e2bAauPjkVnQnMcqGtYZJ4OHEYjrR1ApM0dVnxjcCLahvsEjGkde7TOOC3D8zjrtPbUkRK/0cHYwFK0VrZzqTZhklgAABUISURBVA1X0zAtzW5EqTJytGOREza1Nj+uLNfTgFUbl9l9PMsim7E1+gYJtG1tzdiyaWaTy/6KZC01+QSnGJ+fjxvNwjkPAW2x0eGDSK1bJ4xJsx4hRJ6MjgFoUVBaYDrtW8Q6GhZb4r/HAaytoOUp0o2pMe9coi3sw2r1xQoDVh/SD1eIzNIPrxnpyR55VRcOx24oowPfXKzZzpwvseQeBHqSMhBGx5zaoVf7/1xCp5tomnO++E67CT3s9dsZrb7o8jmfVR2nV9uHKE1GxxgM62rz1ZkMMK5mPus6oaOHeWed+tDPVNjDiGNsgKVIwbOuk6UG9Uda/DYF53yk4BfE3ayO/azY/HnsrwykuGcSx6Z2+CL3mvwipRAie+qoey0Mm3YT0DfbibS3D7cxqessq2bcIwMcdMKY1i+KTpQ6pzbWfmyM/bbJFJl9qy/F+Pr6eBSUlsVY3aEQIqp6taE11yb+l++ctjX8FLbYRIz4wr6kuVbfHPrZDkdkwsFcOEPhzWjFNxcmMUWrLxyf3lwYR9zyHPcnhBCblNExR+zxGP+MVakVhb5V5sWno1vTF15Tn6F06QghVNWrDa25NsakzJQQQoj8HDlypNghbJhyPNZCx1zq6Qkhykd1+qE1hRAbbXJystghCCGEEKJCVN/++W3FjkEIAXR2dhY7BFFiTp06VewQNkw5HmuhYy719IQQ5Uc5ffp0WD/jtttuK1YsQgghhBBCiAKpTpzR2ipjaovCmpqakutqA8h5zk+hz18l/D0q4RjWQ6lfK6WenhBi41S3tbVFJyLtgz/77LNixSMqmFxXmZ09ezbnbQ8dOhT9v5zn/BT6/OWTntPpzHlbl8uV87aJNuKaKpVjXYt8OrmmirnU05NnixDlp3p56TcA1DfcU+RQhBD6gne2pAOvEKKYQl4vLsBpt1O7+tp4vUHsdotu3gy9NYPUL8xiT5dAyIvXVUedx4JF22bGC1gtWFbfqbpf8x4m2heYTbuT9RZiZgYsSQGH8Pb6sXoSz98MvTUH4dwKHgvrZC3npZhxilxVKwUdXlMIIUShraVTZbkPpViOx1romEs9vZleMwGnPlM+AXWzWuZPzTgu96fL9NVir3NRYw6wMKtmGEPeQTiXIZMf2wuzuukLEwGcduMaMqpd9K9XJj/kxbynj8sZV2qkqwsuMJtwbmqxO8Fc00v/iofoolCAxa5zzK5n5jnkZ4IhRhLPy0wv5oAz4ZwWMU6RM4NiMCBDbApRYUIz9JprqKmpocbcy0yo2AEJsXkpilLsEArK4hmB7hrM3uQHS8jbzUT7QnxGNuTFXKM9j2pqqDk4Cpf72KNN7+m7zOhB3fIaM2rSM/T2zqwSTS322RX6l7uJhTNDr35/NTXU1Oyh7/IoB5Pm11Bj9qI/kplesxpvwvzMYdiZXVlhRfc519XI0IJ+3iweT3wmf6ZXi2FPH5fRxdc7Q8g/gWn/2nPPoZkZvL1mamp6iTt7iX8Hbb+mduhOnH9hPyO4oud0PeIUG8NgUAys+UezQl7Mq958BdxObB4hL+bEh1MpplnivN2D1PcvsLKywkI/HOz2FjskITalSsvkq2qxz57DtByMmxvymulmJLlmPUUmeGVhgYXEeee6aOw6x8JKpHbfgmf/hawy3BZnO8v+yFoWPKnSHlpgYWiIc4nLZmNNUULeXi7sn1XjHQFXisJMIVk8K6ysnKOLLs7pCwceI/7lduovpCoAZVZrsWD3zDLUmLgg4e+gnROPPcXfx2Oh1u6JvmVZjzjFxjAohioUQ1XykoQawV6zllGa6VVLdKMHY39UXeY9NNMbLTGae3U3Z4bt1JKiLn2tJL1qLKL8rFbYq7Uzq38lWAi5pFnmhVL77Cx2rR1orWU/XUWOR4jNIDFTX5GZ/Git8EFGte/zSI38nr7LXO7bE/2OT6zxj9YK19RQs2cPe6J5AF3GsN4Y3/7b4mTIFJm4oKupP8goE7Ga6G7Yjz9N3mCG3sF6Ruy11NrruJAhI+qfgGjldG0dTPjXcE7iM7uDi5eZ8M/gNScui8/DzPQOUr+gfkepb0vMeL0uJqjDGslgNw7pCkAFEPJiHqynn+7kNxxpKsaKEqfIW9oa/RnXIGg1gisj+4k2PLN4WFkYorHrXFzJL6LW6GREmz9Sv0x35G7KsJ3FsxIreVo8Wqkxi1hEeVmlkBj7Eoh/yIS8ZmpqzJjN5uTagsSCYcJ0zmlmiLPchLyDLLZbix2GEJtCJHOvz+SHw+F0q5efVLXzKT7nuhppt2bO7XVFc9Qm6tKuWovdY6c2uMxl9sdq6heGaDS10z50Llorb7Hbkyp01Gf9BfaPQLfZSwgLnhW16VG6DG2MhXqWV69YTHlO+jExRD8XYCRxWaziKeQ1c2F/JGMcwmvuhpERYD+z+yEIEApAuzWLjs5ZmumlpnsZkwmM9tmENy1DNHbtT3keNzxOURBp2+hb9ptYPKiVzPcMwrnsakRDfle0hL2nb7QgQeYaiygxqxQS1VeDK0mvG2vts5zrugymfrUpyjkTfa6ZaJrnuhoZGrGnnM41zUxxbqStW7cmfdYi5O2le7m/iKNMCLH5VGwmP29dsWYz57J/zzjTW0PN4CJxj/HgMtRbsRJIzohHKnxqauhmRM1Y19qZHYHu3hki7fpXVvZzYb2amsxcYLHdijHzSvgZwWP0Yq4x453xs9w+gr0WrFYLGAMEZiDkX6Z+lUJT1mH1qu3vV2ad1C/G+kno2+wn16VufJyicAyKYiDlyDsWT7SEunCuncXBLDqlhLx090H/wtpv5IxyiUVUmEbanWpmu9ayn67FQHSJxdnORCSTHvIySH+Wrw3Tp1kqbt68mXE6k5leMy6czMp4Z0JsiMRMfcVm8nUZ6XSfgynr+XSdOFOvkHI/F/avsDLSHr/owiLt1lpqrXAhMZdu8UQraWbt4DXX0OtVR8WJdRoN4e0N4NQ6yab+zgixTH0OFYshvINkMcKPBbu9VnsjMAKDfaivNmqprQX14Mx0L+8vTFOYkJfA/lwqrjY4TlFQBoOhCkNSG/0ZemtiI3XUpiqSLgbUzLbWfj7asqGrXh3TNhTCO5jiRk63HYsEQtp25oPEtswiFrG51dppXxzEG6Iie/9HMvfZZ/JDeM3qa1aP9tT1mnulcCzEBohk7is2kw9xGen0TXdSbbjGGn2Lk4WFFMN06it0aq3UT3SnqJEP4Y32E+xiv93O7Mo50CoKZ2aC2J3QnVCbb22HQCRfEvLn1Owx5O1muT99y4OZmeTGQDO93TCywv4L+nxRkOXRy1BfoIxPrZ24ny8wDSV3iF4YIrEP74bHKQoq/fCajXChO/IqZwJTv+4HEmrt9Jsm1Fc+ewYhMpRWrZ1+JrRtuqG9i8t9e2IXQ7rtAHu/iYk96nbL7UN0Xe6LtYvOFIsoP2kLe7mz95uYcHlxTbTjLFQ+fx3izNVaavIJ+Zm4HD9cXd9lrQ2lEGLdVXQmfx1YPOkyxVqNcZwQXheMRHP/tdhH2pnYo8+waz8CVZ/YXtyCJzLCTiBAqNbO7DkTE/5YTr/W7oHICDLdJI8vn1EIr1ltLpS+0jxE4EJAV/GibhNp/27xnKM+EIr1LVhZYYTuNfUTi3T+jQ4nupZhQtMqfJxiY1SnHlrTgmfWAnjwpNnQ4pllJcXCxPl2uyfjct0CZuM3jO57tVhEGam1028ys6emD2ikayhS2FN/WS/6JqdmFGhkaGEWq9/MwdHLMNpL3YqTgPkg6mRd7BWkxUP7YA0T7Qu6B3seaaaNswzU2pldsRc7CiFEJZrpza7ZDY0MOdMlUcPB0UaGFhLTbGRoIUPGevQgNaPqMzxurVo7s+eWqdljhoVZ7LVqG/zIU7DONMrBmuSYu+rseCyepB96sntWsK8lwxH9sSx1yMlZXXC11nom9tTQp1+/sYu6ENTWRjq1ruDRtpnpPUjfKEwMLbCyos20z7LgNVNTM8hQpl8PjsSvO/b0giwvQux9hVo46rsMjdE/jH5Z4eMUG6M65dCaQqyj1IU9ddzjlM9W+yz6fKsl5UMsBHQl1L7kl2baQqkQQmxWFg8rOT4YLR5tO89K/LM1mzRXq8CweFhZSbdoZX2f5Zliyxh3LfbZ2bg56WKtTfjOyl+kQlcfS7oCQjHjFPlKU6MvRLmI1UIA9PXuL9oIOUKslyNHjhQ7hA1Tjsda6JhLPT0hRPmoTtk+X4iykakWovxMTk4WOwQhhBBCVIjq2z+/rdgxCCGAQ4cOFTsEUWJcLlexQ9gw5XishY651NMTQpQf5fTp03HDA9x2223FikUIIYQQQghRINWJM1pbW4sRh6hgU1NTcl1tADnP+Sn0+auEv0clHMN6KPVrpdTTE0JsnOq2trbohLQPFqJ4xsbGct62s7OzgJGIUpFPJ8pTp04VMJL1V47HWuiYSz09IUT5qV5e+g0A9Q33FDkUIYS+4J0tKaALITZc0E3zid3M+db+y7Hx/NiaJ2kb82HN5YdVg25sndA25yApEr8NGz7yDlGIMlatyPCaQghR0tZSu1ruQymW47EWOuZSTw8gODWOCROKMsl02JecyQ66ae6EsTkHRvzYlBaGgSaXC9M4HJ1zEMvXN1BnBAgSDBoxpsnwB4NBjCkWLpp240u1gdVHm02h+UqAgaU6WoZTp9vkCjDnyKWUIUTpMygGA5UyxKbfpqAoNvyFSCzoprlQaQkhhBCVIuimc2kAn89HONzGpKLQ7A6iZtRjq5k6WrXMvBVfOEw4HGaMJRrGHBj9NhRFQVFaGJ53UqcoKEoddZ1uYkkEcTcr2noKdXV12LL9UvbbUJrd1PnCzDmMWH3q/hM/AVcPHa2SyReVy2BQDJTEj2YF3TRnfQenZvWFcTUVaH9GB3OpailEwRS0YJYrKdAJIcSa+E8sMRBtD2PFFw4wsBsgwIm6DM/ToJsTHMWBm+Z01evzTuqUZtxBACO7TT1Ma5ny6Z4emIxl/BVFQalzMj/cok1HtlM1RQoa0UJF7JNndkOIsmFQDFUohqrkJUE/tmhJuhmbzabdQH5skRvK79bWid3YQXdz7GZqtuEP6pO00awta7bpSu1+W8LNqqDo7sLs08zyzl1lf2oGNDETGjvuZm1fak1D7MGSKU6RrKAFs1y3kwKdEEJkzW9TaBkepkWfcW6eoi7SwL6pgbqUWwZxn4CjDiMYHQz0pE6/yRUgHJ4jXUuatsSa+YCLpp5pbTr9dmq6sQJDmzz0xSaRpkY/iLvzOA0DkRtjjIbFRW2Z+gpuumce5/EldZ1pmHQHtdL6WOwGnDvKlROxDL2x7ihjkdd3DUt0RnLIVl/CzRomHKktyJRm0E3n8QZdmpM457M46kz7g+grvvhMqBVfwEUTJgYCAToWWxjvCBCeNjE+tfqxC1WmglnBC4JSoBNCiMLx25hsCzPd04QrEMto90Sb6KQXdHey1ACd2rNzsi2SWZ+mp8lFQPvuTG4rHytURF4C+G2RpkJZqmsAZ130uZ3uZYIQlSh1G/3gFOOmARzRLvBGHHOJJeUmXGM+dR2rD5/DSHBqnGHdzaQodTiHx5mKZJCmTkRv8jpndndapjSDU+OYBmIdeoyOo6SpJCicnjasRiPQw4DuhKx27IJVC2YFLwhWQIFu69atSR8hhCgKa4oRbAJLLKZcOZ6xdYyjDgdz2jP56JVIhYm+jX58ZYxK33QnEkaAjvFO1pLXN8XV6Ge/nRDlzqAoBgo18o5xtynu9Vjcq7Sgm04nDERqAbK80zKmWULKJc5iWq1gVuiCYF5KpEB38+bNjNNCCFEc2tvPlmFMu7P4ojMGmHIHIbCEqaGOqaU0r9+HW7KorU9V+SiESMVgMFRhSGyjb2ylY/F4rLQcDOK2NWNb7eaztmEaP5G+KUNPgzpObjCI+3iKjNziFa1Jjto/wObPnKaxtYPF47Ha1KD7BGt6I5dqf7la7dhFZmVSECxGgS6SuZdMvhCidMRG0vFZwe/3q/Pihs2MX7+VTpQWaGudgjRNd8LhMGPRH+ENcmUxuenOmgWW4ipopOmO2EzSDK9pxDE2wFJnpFd7J0sNY/gcRm2UEoWW4XmcdVp76mgBwIpvrIHJyHZKc6yttdHBAOPR9OjoYd6pGyrL6GDANK6+vqs7Dh0B7RVh5jTHBpaitcCdS224moZpaXavfuRp9xdpo63gnI88YJpxB/3YtPbe6vHG9qMeR4Y4BZBFwazABcGM2+WqSAU6yeQLIUpXkCvHj6/alCawNE9T0yItdUvsztAZNn6s/FjTnXCuAydYfQmVMz5wy/ez2Byq0w6tabTimwsn/wiF0cFc2JE+RaMD35wj5Y9XWH1zhHULHA5fxuXZpGm0+piL2yhMhuiy2J9aS5Hyxzf08x2x/YSjiaSPU6AVzGx0KgrzQFPPNK6mFlqaGwjPORigGUVxAk24XFpBcLdaW6QWzJqp05b3uPQFwSvYOhVa5gGaaOrpYMyn1Sql3S72Ay4AKMPqfgMDLNU5mQeaGwJ0MKzGN0Asnkz7E0KICmf1jXGlWUGJtr5pwhWINdNt1p6hPdPat6Q2zzQdZs4HkQq1Fl2aw4pTNxVJz4hjLs03atx+wms/hlZoVhTm6WE620yDEGWoOuXQmkKsk0wFs/UoCKbfTgp0QgiRGyOOuTSVaqkqA5PmZXj+Zh3CKpWOVh9zmar/V9teiAqRvka/nOlK+qn0TIeTRw4QQogSdeTIkWKHsGHK8VgLHXOppyeEKB/Vye3zK4CU1EWZmpycLHYIQgghhKgQ1bd/fluxYxBCAJ2dncUOQZSYU6dOFTuEDVOOx1romEs9PSFE+VFOnz4d14vltttuK1YsQgghhBBCiAKpTpzR2tqaaj0hcjY1NSXX1QaQ85yfQp+/Svh7VMIxrIdSv1ZKPT0hxMapbmtri05E2gd/9tlnxYpHVDC5rjI7e/ZsztseOnQo+n85z/kp9PnLJz2n07n6Smm4XK6ct020EddUqRzrWuTTyTVVzKWenjxbhCg/1ctLvwGgvuGeIocihNAXvLMlHXiFEEIIkUq1UonDawohRAVZS6fKch9KsRyPtdAxl3p6QojyUZnDawohhBBCCLHJVeYPZgkhhBBCCLHJVSuGqmLHIIQQQgghhCgwqdEXQgghhBCiAkkbfSFK2NatW5Pm3bx5swiRCCGEEKLc/P8BZd3Exp8SjwAAAABJRU5ErkJggg==)



搜索时要写入数据库，这个行为用户并不关心，可以使用多线程异步的方式来进行写入。



# 12-集成线程池 05:57

```java
package com.heima.common.threadpool;

import lombok.Data;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.ThreadPoolExecutor;

@Data
@Configuration
@EnableAsync //开启异步请求
public class ThreadPoolConfig {

    private static final int corePoolSize = 10;   // 核心线程数（默认线程数）
    private static final int maxPoolSize = 100;   // 最大线程数
    private static final int keepAliveTime = 10;  // 允许线程空闲时间（单位：默认为秒）
    private static final int queueCapacity = 500; // 缓冲队列数

    /**
     * 默认异步线程池
     * @return
     */
    @Bean("taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor(){
        ThreadPoolTaskExecutor pool = new ThreadPoolTaskExecutor();
        pool.setThreadNamePrefix("--------------全局线程池-----------------");
        pool.setCorePoolSize(corePoolSize);
        pool.setMaxPoolSize(maxPoolSize);
        pool.setKeepAliveSeconds(keepAliveTime);
        pool.setQueueCapacity(queueCapacity);
        // 用于被拒绝任务的处理程序，它直接在 execute 方法的调用线程中运行被拒绝的任务；如果执行程序已关闭，则会丢弃该任务
        pool.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 初始化
        pool.initialize();
        return pool;
    }

}
```



# 13-接口定义 10:26

```java
package com.heima.apis.search;

import com.heima.model.article.dtos.UserSearchDto;
import com.heima.model.common.dtos.ResponseResult;

public interface ApUserSearchControllerApi {


    /**
     * 查询搜索历史
     * @param userSearchDto
     * @return
     */
    public ResponseResult findUserSearch(UserSearchDto userSearchDto) ;

    /**
     * 删除搜索历史
     * @param userSearchDto
     * @return
     */
    public ResponseResult delUserSearch(UserSearchDto userSearchDto) ;

}
```

```java
package com.heima.model.search.dtos;

import com.heima.model.common.annotation.IdEncrypt;
import com.heima.model.search.pojos.ApUserSearch;
import lombok.Data;

import java.util.Date;
import java.util.List;


@Data
public class UserSearchDto {
	// 设备ID
    @IdEncrypt
    Integer equipmentId;
    /**
    * 搜索关键字
    */
    String searchWords;
    /**
    * 当前页
    */
    int pageNum;
    /**
    * 分页条数
    */
    int pageSize;
    /**
    * 最小时间
    */
    Date minBehotTime;
    /**
    * 接收搜索历史记录id
    */
   Integer id;

    public int getFromIndex(){
        if(this.pageNum<1)return 0;
        if(this.pageSize<1) this.pageSize = 10;
        return this.pageSize * (pageNum-1);
    }
    
}
```



# 14-查询搜索列表 05:38

```java
@Override
public ResponseResult findUserSearch(UserSearchDto dto) {
    //获取每页数量超过50条，直接返回参数错误
    if(dto.getPageSize()>50){
        return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
    }
    //通过用户和设备ID，获取entryId
    ResponseResult ret = getEntryId(dto);
    if(ret.getCode()!=AppHttpCodeEnum.SUCCESS.getCode()){
        return ret;
    }
    IPage pageParam = new Page(0,dto.getPageSize());
    QueryWrapper<ApUserSearch> queryWrapper = new QueryWrapper();
    //只查询entryId匹配的，并且有效的数据
    queryWrapper.lambda().eq(ApUserSearch::getEntryId,ret.getData()).eq(ApUserSearch::getStatus,1);
    IPage<ApUserSearch> page = page(pageParam,queryWrapper);
    return ResponseResult.okResult(page.getRecords());
}

    public ResponseResult getEntryId(UserSearchDto dto){
        ApUser user = AppThreadLocalUtils.getUser();
        // 用户和设备不能同时为空
        if(user == null && dto.getEquipmentId()==null){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_REQUIRE);
        }
        Integer userId = null;
        if(user!=null){
            userId = user.getId();
        }
        ApBehaviorEntry apBehaviorEntry = behaviorFeign.findByUserIdOrEntryId(userId,dto.getEquipmentId());
        // 行为实体找以及注册了，逻辑上这里是必定有值得，除非参数错误
        if(apBehaviorEntry==null){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }
        return ResponseResult.okResult(apBehaviorEntry.getId());
    }
```



# 15-删除和新增搜索记录 09:20

```java
@Override
public ResponseResult delUserSearch(UserSearchDto dto) {
    if(dto.getHisList() ==null || dto.getHisList().size()<=0){
        return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_REQUIRE);
    }
    ResponseResult ret = getEntryId(dto);
    if(ret.getCode()!=AppHttpCodeEnum.SUCCESS.getCode()){
        return ret;
    }
    //获取批量的删除列表
    List<Integer> ids = dto.getHisList().stream().map(ApUserSearch::getId).collect(Collectors.toList());
    //将这些删除列表中的数据设置status为0
    boolean update = update(Wrappers.<ApUserSearch>lambdaUpdate().eq(ApUserSearch::getEntryId, ret.getData()).in(ApUserSearch::getId, ids).set(ApUserSearch::getStatus, 0));
    return ResponseResult.okResult(update);
}
```

```java
 @Override
    @Async("taskExecutor")
    public void insert(Integer entryId,String searchWords) {
        //查询生效的记录是否存在
//        int count = count(Wrappers.<ApUserSearch>lambdaQuery().eq(ApUserSearch::getEntryId, entryId).eq(ApUserSearch::getKeyword, searchWords));
        ApUserSearch userSearch = getOne(Wrappers.<ApUserSearch>lambdaQuery().eq(ApUserSearch::getEntryId, entryId).eq(ApUserSearch::getKeyword, searchWords));

        if(userSearch != null && userSearch.getStatus() == 1){
            log.info("当前关键字已存在，无须再次保存");
            return ;
        }else if (userSearch != null && userSearch.getStatus() == 0){
            userSearch.setStatus(1);
            updateById(userSearch);
            return;
        }
        //保存新的关键字
        ApUserSearch apUserSearch = new ApUserSearch();
        apUserSearch.setEntryId(entryId);
        apUserSearch.setKeyword(searchWords);
        apUserSearch.setStatus(1);
        apUserSearch.setCreatedTime(new Date());
        save(apUserSearch);
    }
```



# 16-测试 12:48

esArticleSearch方法中：

```java
//如果是翻页到后面几页，就不要存储搜索内容了
if(dto.getFromIndex()==0){
    ResponseResult result = apUserSearchService.getEntryId(dto);
    if(result.getCode()!=AppHttpCodeEnum.SUCCESS.getCode()){
        return result;
    }
    apUserSearchService.insert((int)result.getData(),dto.getSearchWords());
}
```



# 17-关键字联想词 10:53

![1587369167442](data:image/jpg;base64,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)

```java
package com.heima.search.service.impl;

import com.baomidou.mybatisplus.core.metadata.IPage;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.heima.model.common.dtos.ResponseResult;
import com.heima.model.common.enums.AppHttpCodeEnum;
import com.heima.model.search.dtos.UserSearchDto;
import com.heima.model.search.pojos.ApAssociateWords;
import com.heima.search.mapper.ApAssociateWordsMapper;
import com.heima.search.service.ApAssociateWordsService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

/**
 * <p>
 * 联想词表 服务实现类
 * </p>
 *
 * @author itheima
 */
@Slf4j
@Service
public class ApAssociateWordsServiceImpl extends ServiceImpl<ApAssociateWordsMapper, ApAssociateWords> implements ApAssociateWordsService {

    @Override
    public ResponseResult searchAssociate(UserSearchDto dto) {
        if(dto.getPageSize()>50 || dto.getPageSize() < 1){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }
        IPage pageParam = new Page(0,dto.getPageSize());
        //模糊查询ap_associate_words表
        IPage page = page(pageParam, Wrappers.<ApAssociateWords>lambdaQuery().like(ApAssociateWords::getAssociateWords, dto.getSearchWords()));
        return ResponseResult.okResult(page.getRecords());
    }
}
```



# 18-优化改造关键字联想词方案说明 07:35

优化方案：

- 数据能够缓存到redis
- 构造Trie树数据结构，高效查询数据

Trie树：又称单词查找树，Trie树，是一种树形结构，是一种哈希树的变种。典型应用是用于统计，排
序和保存大量的字符串（但不仅限于字符串），所以经常被搜索引擎系统用于文本词频统计。它的优点
是：利用字符串的公共前缀来减少查询时间，最大限度地减少无谓的字符串比较，查询效率比哈希树
高。

**词组为：黑马程序员、黑马头条、黑马旅游、黑猫**

![1601794912789](data:image/jpg;base64,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)

- 根节点不包含字符，除根节点外的每一个子节点都包含一个字符。
- 从根节点到某一节点，路径上经过的字符连接起来，就是该节点对应的字符串。
- 每个单词的公共前缀作为一个字符节点保存。



# 19-搭建redis环境 04:46

拉取镜像

```
docker pull redis
```

创建容器并设置开机自启

```
docker run -d --name redis --restart=always -p 6379:6379 redis --requirepass "1234qwer"
```

在common模块中集成redis

新建redis.properties文件

```properties
#redis config
spring.redis.host=localhost
spring.redis.port=6379
#spring.redis.password=123456
#spring.redis.password=1234qwer
spring.redis.timeout=90000
#连接池的最大数据库连接数
spring.redis.lettuce.pool.max-active=8
#连接池的最大空闲数
spring.redis.lettuce.pool.max-idle=8
#连接池的最大建立连接等待时间  -1 为无限等待
spring.redis.lettuce.pool.max-wait=-1
#连接池的最大空闲数  0 表示不限制
spring.redis.lettuce.pool.min-idle=0
```

新建配置类

```java
package com.heima.common.redis;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;

@Configuration
@PropertySource("classpath:redis.properties")
@ConfigurationProperties(prefix = "spring.redis")
public class RedisConfiguration {
    
}
```

(3)在搜索微服务中集成redis

```java
package com.heima.search.config;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan("com.heima.common.redis")
public class RedisConfig {
}
```



# 20-业务层 09:19

```java
@Override
public ResponseResult searchV2(UserSearchDto dto) {
    //1.从缓存中获取数据
    String assoStr = redisTemplate.opsForValue().get("associate_list");
    List<ApAssociateWords> apAssociateWords = null;
    if(StringUtils.isNotEmpty(assoStr)){
        //2.缓存中存在，直接拿数据 "[{},{}]"
        apAssociateWords = JSON.parseArray(assoStr, ApAssociateWords.class);
    }else {
        //3.缓存中不存在，从数据库中获取数据，存储到redis
        apAssociateWords = list();
        redisTemplate.opsForValue().set("associate_list", JSON.toJSONString(apAssociateWords));
    }
    //4.构建trie数据结构，从trie中获取数据，封装返回
    Trie t = new Trie();
    for (ApAssociateWords apAssociateWord : apAssociateWords) {
        t.insert(apAssociateWord.getAssociateWords());
    }

    List<String> ret = t.startWith(dto.getSearchWords());
    List<ApAssociateWords> resultList  = new ArrayList<>();
    for (String s : ret) {
        ApAssociateWords aaw = new ApAssociateWords();
        aaw.setAssociateWords(s);
        resultList.add(aaw);
    }

    return ResponseResult.okResult(resultList);
}
```



使用es进行模糊查询，替代当前方案

# 21-测试 04:15

简化测试步骤，可以直接将controller调用的service方法修改一下