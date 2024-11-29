# Elasticsearch
## Elasticsearch 同义词
### 1、读取本地文件

在elasticsearch安装目录下的config目录下（elasticsearch-7.2.0/config）新建synonyms.txt文本；

并在文本内添加同义词如（英文逗号分隔）:
```
美元,美金,美币
苹果,iphone
```

创建索引，配置同义词过滤
```
PUT /my_index
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "my_tokenizer": {
          "type": "standard"
        }
      },
      "filter": {
        "my_synonym_filter": {
          "type": "synonym",
          "synonyms_path": "analysis/synonyms.txt"  
        }
      },
      "analyzer": {
        "my_synonym_analyzer": {
          "type": "custom",
          "tokenizer": "my_tokenizer",
          "filter": [
            "lowercase",         
            "my_synonym_filter"  
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "my_synonym_analyzer"  
      }
    }
  }
}
```
### 2、远程文件地址或者远程接口服务

表结构
```
CREATE TABLE `dynamic_synonym_words` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `words` varchar(500) COLLATE utf8mb4_general_ci NOT NULL,
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `creator` bigint NOT NULL COMMENT '创建人',
  `update_time` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `updater` bigint DEFAULT NULL COMMENT '更新人',
  `deleted` tinyint(1) NOT NULL DEFAULT '0' COMMENT '删除标志',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='同义词';
```
测试数据添加
```
INSERT INTO `njydzq`.`dynamic_synonym_words` (`id`, `words`, `create_time`, `creator`, `update_time`, `updater`, `deleted`) VALUES (1, '苹果,iPhone,iPhone16', '2024-11-28 19:16:55', 1, '2024-11-29 15:20:05', 1, 0);
INSERT INTO `njydzq`.`dynamic_synonym_words` (`id`, `words`, `create_time`, `creator`, `update_time`, `updater`, `deleted`) VALUES (2, '番茄,西红柿,圣女果', '2024-11-29 09:01:08', 1, '2024-11-29 09:03:14', 1, 0);
INSERT INTO `njydzq`.`dynamic_synonym_words` (`id`, `words`, `create_time`, `creator`, `update_time`, `updater`, `deleted`) VALUES (3, '汉堡,肯德基,KFC', '2024-11-29 15:20:47', 1, '2024-11-29 15:20:49', 1, 0);
```
远程接口服务

```
    @RequestMapping(value = "/word", method = {RequestMethod.GET, RequestMethod.HEAD}, produces = "text/html;charset=UTF-8")
    public String getSynonymWord(HttpServletResponse response) {
        StringBuilder words = new StringBuilder();
        List<DynamicSynonymWords> dynamicSynonymWords = dynamicSynonymWordsMapper.selectList(new LambdaQueryWrapper<DynamicSynonymWords>()
                .orderByDesc(DynamicSynonymWords::getUpdateTime));

        dynamicSynonymWords.stream().findFirst().ifPresent(dynamicSynonymWord -> {
            response.setHeader("Last-Modified", dynamicSynonymWord.getUpdateTime().toString());
            response.setHeader("ETag", String.valueOf(dynamicSynonymWord.getUpdateTime().getTime()));
        });

        List<DynamicSynonymWords> synonymWords = dynamicSynonymWords.stream().filter(dynamicSynonymWord -> dynamicSynonymWord.getDeleted() == 0)
                .collect(Collectors.toList());

        for (DynamicSynonymWords dynamicSynonymWord : synonymWords) {
            words.append(dynamicSynonymWord.getWords()).append("\n");
        }
        return words.toString();
    }
```
创建索引
```
PUT /my_index2
{
  "settings": {
    "index.max_ngram_diff": 40,
    "analysis": {
      "analyzer": {
        "default": {
          "tokenizer": "ik_max_word"
        },
        "pinyin": {
          "tokenizer": "pinyin"
        },
        "ngram": {
          "tokenizer": "ngram"
        },
        "synonym_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": [
            "lowercase", 
            "remote_synonym"
          ]
        }
      },
      "filter": {
        "remote_synonym": {
          "type": "dynamic_synonym",
          "synonyms_path": "https://xxx/server/api/sys/word",
          "interval": 30
        }
      },
      "tokenizer": {
        "pinyin": {
          "type": "pinyin",
          "keep_first_letter": true,
          "keep_separate_first_letter": false,
          "keep_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "lowercase": true
        },
        "ngram": {
          "type": "ngram",
          "min_gram": 1,
          "max_gram": 40,
          "token_chars": [
            "letter",
            "digit",
            "punctuation",
            "symbol",
            "whitespace"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "caseId": {
        "type": "long"
      },
      "title": {
        "type": "keyword",
        "fields": {
          "ik_max_word": {
            "type": "text",
            "analyzer": "ik_max_word"
          },
          "pinyin": {
            "type": "text",
            "analyzer": "pinyin"
          },
          "ngram": {
            "type": "text",
            "analyzer": "ngram"
          },
          "synonym": {
            "type": "text",
            "analyzer": "synonym_analyzer"  
          }
        }
      }
    }
  }
}
```
### 3、同义词插件连接数据库管理词库
重写elasticsearch-analysis-dynamic-synonym插件
```

```


**注：** 同义词新增或修改只对增量数据有效，存量数据需要重新索引才能应用同义词规则
