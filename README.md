# XlsxConvert

[![License](https://img.shields.io/badge/License-Apache%202.0-brightgreen.svg)](LICENSE) [![HitCount](http://hits.dwyl.io/VyronLee/XlsxConvert.svg)](http://hits.dwyl.io/VyronLee/XlsxConvert)

一个用于把 xls/xlsx 文件转换成 Lua， Json 或 ProtoBuf 格式的工具。



目录
=================

   * [XlsxConvert](#xlsxconvert)
   * [特性](#特性)
   * [预览](#预览)
   * [安装](#安装)
   * [用法](#用法)
      * [格式转换](#格式转换)
      * [配置加载以及使用](#配置加载以及使用)
   * [License](#license)



# 特性

使用该工具生成的配置表具有以下特点：

- 可根据输入进行配置列过滤

  该工具可供用户输入一个“配置列过滤正则表达式”，只有符合表达式的列数据才会实际生成到 Json 文件中。该功能在客户端与服务端使用同一套配置表时会十分方便，仅仅输出实际需要的字段，避免数据冗余。

- 数据查询速度快

  该工具转换格式时要求用户提供一个“索引字段列表”，工具会根据设定的索引字段预先生成 key-values 映射关系，能够极大减小运行时数据查询的时间复杂度。而且索引字段支持单个键值索引或者多个键值索引，键值不设个数限制，可根据实际需求进行配置。

此外，对于生成的 Lua 格式：

- 文件体积小

  生成的Lua文件把配置表 Header 与 Content 分离，Header 字段只会出现一次，避免当文件成千上万行时Header 字段大量重复的问题；Content 每行以类似“数组”的形式保存，可以方便地与原表格进行人工校验。

- 数据读取方便

  生成的 Lua 文件中提供一个统一的数据获取接口`getData`，不管是使用单个 key ，或者使用多个 key 进行数据获取，都可以以单一接口实现。



预览
=========

* 生成的 Json 文件样式:

  ```json
  {
      "data": [
          {
              "id": 1,
              "hp": 100,
              "mp": 50,
              "attack": 10,
              "defend": 10
          },
          {
              "id": 2,
              "hp": 80,
              "mp": 200,
              "attack": 12,
              "defend": 8
          },
          {
              "id": 3,
              "hp": 80,
              "mp": 100,
              "attack": 15,
              "defend": 5
          }
      ],
      "index": {
          "id": {
              "1": [
                  1
              ],
              "2": [
                  2
              ],
              "3": [
                  3
              ]
          }
      }
  }
  ```

  data 部分为表格数据存储；index 部分为快速索引存储，可用于快速查询数据。

* 生成的 Lua 文件样式：

  ```Lua
  ------------------------------------------------------------
  --      @file ActorConf_Properties.lua
  --     @brief Auto generated by XlsxConverter, DO NOT EDIT!
  --    @author VyronLee(lwz_jz@hotmail.com)
  -- @Copyright Copyright(c) 2019, Apache-2.0
  ------------------------------------------------------------
  local keys = {
    ['id'] = {index=1,type="int",brief="职业ID"},
    ['hp'] = {index=2,type="int",brief="初始HP"},
    ['mp'] = {index=3,type="int",brief="初始MP"},
    ['attack'] = {index=4,type="int",brief="初始攻击"},
    ['defend'] = {index=5,type="int",brief="初始防御"},
  }
  
  local mt = {
    __index = function(t,k)
      return keys[k] and t[keys[k].index]
    end,
    __tostring = function(t)
      local ret = {}
      for k in pairs(keys) do
        if type(t[k]) == "number" then
          table.insert(ret, ("%s = %s"):format(k, t[k]))
        elseif type(t[k]) == "string" then
          table.insert(ret, ("%s = '%s'"):format(k, t[k]))
        end
      end
      return ("{%s}"):format(table.concat(ret, ", "))
    end,
  }
  local __mt = setmetatable
  
  local conf = {}
  
  conf.data = {
    [1] = __mt({1,100,50,10,10,}, mt),
    [2] = __mt({2,80,200,12,8,}, mt),
    [3] = __mt({3,80,100,15,5,}, mt),
  }
  
  conf.indexes = {
    ['id'] = {
      ['1'] = {1},
      ['2'] = {2},
      ['3'] = {3},
    },
  }
  
  local printable = {
    __tostring = function(t)
      local ret = {}
      for _, v in ipairs(t) do
        table.insert(ret, tostring(v))
      end
      return table.concat(ret, "\n")
    end
  }
  
  conf.parseArgs = function(self, ...)
    local keys, values = {}, {}
    local args = { ... }
    for idx,val in ipairs(args) do
      if idx % 2 == 1 then
        keys[#keys + 1] = val
      else
        values[#values + 1] = val
      end
    end
    return keys, values
  end
  
  conf.getData = function(self, ...)
    local ret
    if select("#", ...) > 0 then
      local keys, values = self:parseArgs(...)
      local keyHash = table.concat(keys, '-')
      local valueHash = table.concat(values, '-')
      local keyMap = self.indexes[keyHash] or {}
      local valueMap = keyMap[valueHash] or {}
      ret  = {}
      for idx,val in ipairs(valueMap) do
        ret[#ret + 1] = self.data[val]
      end
    else
      ret = self.data
    end
    return __mt(ret, printable)
  end
  
  return conf
  ```

  同样是包含数据表 data 以及索引表 indexes，以及几个辅助函数。

* 生成的 Protobuf 文件样式：

  这个没法预览，二进制格式数据文件。一般包含有两个，如：

  `ActorConf_Properties.dat.pb` 以及 `ActorConf_Properties.idx.pb`

  其中 `dat.pb` 为数据文件，`idx.pb` 为索引文件。



安装
============

本工具依赖 Python 3.6 及以上版本，以及 Pip。确认依赖安装好后，执行以下命令：

``` shell
git clone https://github.com/VyronLee/XlsxConverter.git
cd XlsxConverter
pip install -e .
```



用法
=====

## 格式转换

示例：

``` python
import xlsx_converter

conf = {                        # header config.
    "header_row_count": 4,
    "key_row_index":    0,
    "type_row_index":   1,
    "filter_row_index": 2,
    "brief_row_index":  3,
}
ip = "./input/EquipmentConf.xlsx"    # input file path.
op = "./output"                      # output directory.
re = ".*c+"                          # just filter out cols contains 'c'
idx = [["id"], ["id", "quality"]]    # indexer list.
out_format = "pb"                    # output format, one of "json", "lua", "pb"
options = {                          # options, required when out_format is "pb"
     "generate_codes": True,
     "codes_type": "csharp",         # generate code type, one of "cpp", "csharp", "java", 
}                                    # "js", "objc", "php", "python" or "ruby".

xls_convert.convert(conf=conf, ip=ip, op=op, filter_re=re, indexers=idx, out_format=out_format, options=options)
```

## 配置加载以及使用

* Json

  这个是标准的 Json 格式文件，使用各语言对应的 Json 库解析即可。

* Lua

  ```Lua
  local equipment_conf_path = "EquipmentConf_Properties"
  local equipment_conf = require(equipment_conf_path)
  
  -- 查询所有
  local all_equipment_conf = equipment_conf:getData()
  print("Dump EquipmentConf.Properties: ")
  print(all_equipment_conf)
  
  -- 两个 Key 查询
  for i=1, 4 do
      print(("Query record with 'id': %s and 'quality': 1"):format(i))
      print(equipment_conf:getData("id", i, "quality", 1)[1])   -- 输出匹配到的第一条配置
  end
  
  -- 单个 Key 查询
  print("Query record with 'quality': 1")
  print(equipment_conf:getData("quality", 1)) -- 输出匹配到的所有配置
  ```

* Protobuf，以 csharp 为例：

  ```csharp
  using System;
  using System.Collections.Generic;
  using System.IO;
  using Google.Protobuf;
  using XlsxConvert.Auto.ActorConf;
  using XlsxConvert.Auto.EquipmentConf;
  using XlsxConvert.Auto.Indexers;
  
  namespace csharp
  {
      internal static class Program
      {
          private const string DataDir = "../../../../../..";
          private const string EquipmentConf_SheetPath = DataDir + "/output/EquipmentConf_Properties.dat.pb";
          private const string EquipmentConf_IndexPath = DataDir + "/output/EquipmentConf_Properties.idx.pb";
  
          public static void Main()
          {
              // 输出所有配置
              Console.WriteLine("Dump EquipmentConf.Properties:");
              var equipmentSheet = LoadAndParseFromFile<EquipmentConf_Properties_Sheet>(EquipmentConf_SheetPath);
              foreach (var record in equipmentSheet.Data)
                  Console.WriteLine(record);
              
              // 使用两个 Key 查询
              indexes = LoadAndParseFromFile<XlsxRecordIndexes>(EquipmentConf_IndexPath);
              for (var i = 0; i < 4; i++)
              {
                  var idx = GetRecordIndex(indexes, "id", i + 1, "quality", 1);
                  Console.WriteLine("Query record with 'id': {0} and 'quality': 1", i + 1);
                  if (idx >= 0)
                      Console.WriteLine(equipmentSheet.Data[idx]);
                  else
                      Console.WriteLine("Not found!");
              }
  
              // 使用单个 Key 查询
              Console.WriteLine("Query record with 'quality': 1");
              var idxes = GetRecordIndexes(indexes, "quality", 1);
              if (idxes.Count > 0)
                  foreach (var idx in idxes)
                      Console.WriteLine(equipmentSheet.Data[idx]);
          }
  
          // 加载以及解析 Protobuf 文件
          private static T LoadAndParseFromFile<T>(string path) where T: IMessage<T>, new()
          {
              var bytes = File.ReadAllBytes(path);
              var obj = new MessageParser<T>(() => new T()).ParseFrom(bytes);
              return obj;
          }
  
          // 使用单个 Key 查询索引, 返回符合条件的第一个
          private static int GetRecordIndex<T>(XlsxRecordIndexes indexes, string key, T value)
          {
              var ids = GetRecordIndexes(indexes, key, value);
              if (ids == null)
                  return -1;
  
              return ids.Count > 0 ? ids[0] : -1;
          }
        
          // 使用单个 Key 查询索引, 返回所有符合条件的索引
          private static IList<int> GetRecordIndexes<T>(XlsxRecordIndexes indexes, string key, T value)
          {
              if (!indexes.Values.TryGetValue(key, out var values))
                   throw new KeyNotFoundException("No key defined in sheet: " + key);
  
              if (values.Values.TryGetValue(value.ToString(), out var ids))
                  return ids.Values;
  
              return null;
          }
  
          // 使用两个 Key 查询索引, 返回符合条件的第一个
          private static int GetRecordIndex<T1, T2>(XlsxRecordIndexes indexes, string key1, T1 value1, string key2, T2 value2)
          {
              var ids = GetRecordIndexes(indexes, key1, value1, key2, value2);
              if (ids == null)
                  return -1;
  
              return ids.Count > 0 ? ids[0] : -1;
          }
  
          // 使用两个 Key 查询索引, 返回所有符合条件的索引
          private static IList<int> GetRecordIndexes<T1, T2>(XlsxRecordIndexes indexes, string key1, T1 value1, string key2, T2 value2)
          {
              var keyHash = $"{key1}-{key2}";
              if (!indexes.Values.TryGetValue( keyHash, out var values))
                   throw new KeyNotFoundException("No key defined in sheet: " +  keyHash);
  
              var valueHash = $"{value1.ToString()}-{value2.ToString()}";
              if (values.Values.TryGetValue(valueHash, out var ids))
                  return ids.Values;
  
              return null;
          }
  
      }
  }
  ```

可查看 example 获得更详细的用法。




License
=======

Apache License 2.0.

