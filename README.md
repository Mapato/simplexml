simplexml
=========

Transforming xml to a python dictionary of lists

Sorry, it is nearly impossible to commit the source file, so I cannot share my work.

from xml.sax.saxutils import escape
import xml.sax.handler
import os

class TreeBuilder(xml.sax.handler.ContentHandler):
    def __init__(self,keyattr=[],contentkey='content',delimiter='_'):
        self.stack = []
        self.root = {}
        self.keyattr = keyattr
        self.contentkey = contentkey
        self.delimiter = delimiter
        
    def set_keyattr(self,keyattr=[]):
        self.keyattr = keyattr
    def set_contentkey(self,contentkey='content'):
        self.contentkey = contentkey
    def set_delimiter(self,delimiter='_'):
        self.delimiter = delimiter
        
    def startElement(self, name, attrs):
        c1 = self.keyattr
        if attrs:
            c3 = filter(lambda x: x in c1, attrs.keys())
            c2 = list(set(attrs.keys())-set(c1))
            if len(c3) > 0:
                ka = c3[0]
                key = ka + self.delimiter + attrs[ka]
                data = {key:{ k:attrs[k] for k in c2}}
            else:
                data = [{ k:v for k, v in attrs.items()}]
                key = '0'
        else:
            data = []
            key = '0'
        self.stack.append((name, data, '', key))
        
    def endElement(self, name):
        (s_name,s_data,s_text,s_key) = self.stack.pop()

        if len(self.stack) == 0:
            self.root = {name:s_data}
        else:
            (p_name,p_data,p_text,p_key) = self.stack.pop()
# simple text entities <tag attr="property">text</tag> are treated here
# if attr is an id-like key this will result in ['tag'] ['property'] = text
# if attr is not id-like it will result in two lines ['tag'][0]['attr'] = property (already filled at the startElement function)
#                                                and ['tag'][0]['content'] = text
            if isinstance(s_data,dict):
                if len(s_data[s_key].keys()) == 0 and s_text != '':
                    s_data[s_key] = s_text
                elif len(s_data[s_key].keys()) > 0 and s_text != '':
                    print s_data[s_key]
                    s_data[s_key].update({self.contentkey:s_text})
            else:    
                if len(s_data) == 0:
                    s_data.append(s_text)
                elif len(s_data) > 0 and s_text != '':
                    s_data[-1].update({self.contentkey:s_text})
                
            if isinstance(p_data, dict):
                if not p_key in p_data:
                    p_data[p_key] = {s_name:s_data}
                else:
                    if not s_name in p_data[p_key]:
                        p_data[p_key].update({s_name:s_data})
                    else: 
                        p_data[p_key][s_name].extend(s_data)
            else:
                if len(p_data)>0 and s_name in p_data[-1]:
                    if isinstance(p_data[-1][s_name],list):
                        p_data[-1][s_name].extend(s_data)
                    else: 
                        p_data[-1][s_name].update(s_data)
                elif len(p_data) > 0 and len(s_data) > 0:
                    p_data[-1].update({s_name:s_data})
                else:
                    p_data.append({s_name:s_data})

            self.stack.append((p_name,p_data,p_text,p_key))
    
    def characters(self, content):
        if content.strip() != '':
            (name,data,text,s_key) = self.stack.pop()
            text = text + content
            self.stack.append((name,data,text,s_key))

class simplexml(object):
    """
A simple class to convert xml into a list of dictionaries
The class mimics the Perl module SimpleXML (http://search.cpan.org/~grantm/XML-Simple-2.20/lib/XML/Simple.pm)
Synopsis
--------
from simplexml import simplexml
sx = simplexml('path-to-xml-file-or-xml-string')
print sx
print sx.XmlOut()
sx.set_KeyAttr(['idref','lang'])
print sx
print sx.XmlOut()
    """
    def __init__(self,source='',keyattr=['id'],contentkey='content',delimiter='_',shortentry=False):
        self.keyattr = keyattr
        self.contentkey = contentkey
        self.delimiter = delimiter[0]
        self.shortentry = shortentry
        self.builder = TreeBuilder(self.keyattr)
        self.source = source
        self.root = {}
        if self.source != '':
            self.XmlIn()

    def __str__(self):
        def flatlist(tree):
            def unfoldelement (path,element):
                if isinstance(element,dict):
                    return "".join(["%s" % unfoldelement(path+'{'+key+'}',element[key]) for key in element.keys()])
                if isinstance(element,list):
                    return "".join(["%s" % unfoldelement(path+'['+str(i)+']',subelement) for i, subelement in enumerate(element)])
                if isinstance(element,(str,basestring,unicode)):
                    return "%s : %s\n" % (path,element)
            return unfoldelement("root>",tree)
        return flatlist(self.root)
        
    def XmlIn(self,source=''):
        if source != '':
            self.source = source
        if os.path.isfile(self.source): 
            xml.sax.parse(self.source, self.builder)
        else:
            xml.sax.parseString(self.source, self.builder)
        self.root = self.builder.root
        
    def XmlOut(self):
        html_escape_table = {
                                '"': "&quot;",
#                                "'": "&apos;",
                            }

        def xmlyfy(tree):
            def extract_list(sublist, tag, depth):
                pre = "\n" + "  " * depth
                global xml_ret
                if isinstance(sublist,dict):
                    for key in sorted(sublist.keys()):
                        (ka,kv) = key.split(self.delimiter)
                        if isinstance(sublist[key], str) or isinstance(sublist[key], unicode):
                            xml_ret = xml_ret +  pre + "<" + tag + " "+ka+"=\""+kv+"\">" + escape(sublist[key],html_escape_table) + "</" + tag + ">"
                        else:
                            extract_subtree(sublist[key],tag,depth,ka,kv)
                else:
                    for item in sublist:
                        if isinstance(item, str) or isinstance(item, unicode):
                            xml_ret = xml_ret +  pre + "<" + tag + ">" + escape(item,html_escape_table) + "</" + tag + ">"
                        else:
                            extract_subtree(item,tag,depth)
                return
               
            def extract_subtree(subtree, tag, depth, ka = '', kv = ''):
                pre = "\n" + "  " * depth
                global xml_ret
                content = ''
                xml_ret = xml_ret +  pre + "<" + tag
                if ka != '':
                    xml_ret = xml_ret + " " + ka + "=\"" + kv + "\""
                has_content = False    
                for k in sorted(subtree.keys()):
                    if isinstance(subtree[k],str) or isinstance(subtree[k], unicode):
                        if k != self.contentkey:
                            xml_ret = xml_ret + " " + k + "=\"" + subtree[k] + "\""
                        else:
                            content = subtree[k]
                    if isinstance(subtree[k],dict) or isinstance(subtree[k], list):
                        has_content = True
        
                if content != '':
                    xml_ret = xml_ret + ">" + escape(content,html_escape_table) + "</" + tag + ">"
                    return
                if has_content or not self.shortentry:
                    xml_ret = xml_ret + ">"
                for k in sorted(subtree.keys()):
                    if isinstance(subtree[k],dict):
                        content = pre
                        extract_list(subtree[k], k, depth+1)
                    if isinstance(subtree[k],list):
                        content = pre
                        if len(subtree[k]) == 0 :
                            if self.shortentry: 
                                xml_ret = xml_ret +  pre + "  <" + k + " />"
                            else:
                                xml_ret = xml_ret +  pre + "  <" + k + "></" + k + ">"
                        else:
                            extract_list(subtree[k], k, depth+1)
                if content == '' and self.shortentry:
                    xml_ret = xml_ret + " />"
                else:            
                    xml_ret = xml_ret + content + "</" + tag + ">"
                return
            
            global xml_ret, keyattr
            xml_ret = ''
            for k in sorted(tree):
                extract_list(tree[k], k, 0)
            return xml_ret
        
        return xmlyfy(self.root)
    
    def set_KeyAttr(self,keyattr=['id']):
        self.keyattr = keyattr
        self.builder.set_keyattr(self.keyattr)
        self.XmlIn()
        
    def set_ContentKey(self,contentkey='content'):
        self.contentkey = contentkey
        self.builder.set_contentkey(self,contentkey)
        self.XmlIn()

    def set_KeyDelimiter(self,delimiter='_'):
        self.delimiter = delimiter[0]
        self.builder.set_delimiter(self,self.delimiter)
        self.XmlIn()

    def set_ShortEntry(self,shortentry=False):
        self.shortentry = shortentry
    
