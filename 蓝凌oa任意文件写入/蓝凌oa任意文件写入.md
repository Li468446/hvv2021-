漏洞名称：蓝凌oa任意文件写入
产品介绍：协同管理OA软件
漏洞细节：
漏洞在/sys/search/sys_search_main/sysSearchMain.do
method 为 editrParam。参数为 FdParameters，在 com.landray.kmss.sys.search.jar 中的 com.landray.kmss.sys.search.actions.SysSearchMainAction 类。method 为 editrParam。
对 fdParemNames 的内容进行了判空。如果不为空,进入 SysSearchDictUtil.getParamConditionEntry 方法。
也是对 fdParemNames 进行了一次判空。然后传入 ObjectXML.objectXMLDecoderByString 方法。
将传入进来的 string 字符进行替换。将其载入字节数组缓冲区，在传递给 objectXmlDecoder。
在 objectXmlDecoder 中。就更明显了。典型的 xmlDecoder 反序列化。
整体流程只对 FdParameters 的内容进行了一些内容替换。 导致 xmlDecoder 反序列化漏洞。
利用方式：
Xmldecoder payload 生成
https://github.com/mhaskar/XMLDecoder-payload-generator