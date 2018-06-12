# get_time.web_data_mongo



      # -*- coding: utf-8 -*-
      from lxml import etree
      import sys
      import urllib
      import json
      from pymongo import MongoClient
      from selenium import webdriver
      reload(sys)
      sys.setdefaultencoding('utf-8')

'''
=======================================================
data_info_dict:作为总字典，储存电影id，名字，年份等基本信息  
                并会传给各个程序以便储存相应的信息          
                在程序的最后将其写入文档                   
                
get_find_url:将文件中的电影名和id取出并存到总字典中，并获取到
相应的电影搜索链接传给下一个程序

get_right_url:在电影搜索页中匹配出相应的电影页面链接，传给其他
程序获取信息

get_actors_infor:获取演员名和个人链接并存成字典，最后把字典存
进总字典中
get_worker_list：获取其他工作人员的姓名和个人链接并存成字典，
最后把字典存进总字典中
=========================================================
'''






'''进入搜索页面'''

      def get_find_url():
  
  
          a = open('./data/2010.txt', 'r').readlines()
          year = '2010'
          data = 1
          data_info_dict = {}
          for line in a:
              e = line.replace('', '\t').split('\t',)
              '''把电影名（字符串形式）转成ｕｒｌ编码'''
              # print urllib.quote(e[2])
              # print e[2]
              info = 0
              movie_name = e[0]
              movie_id = e[1].replace('\n','')
              url = 'http://search.mtime.com/search/?q='+urllib.quote(e[0])+year
              data_info_dict['_id'] = movie_id
              data_info_dict['year'] = year
              data_info_dict['movie_name'] = movie_name
              print '==============='
              print data
              data += 1
              # print url
              # print movie_name
              # print movie_id
              '''
              断点重续，跳过已经爬完的电影进入下一个电影
              '''
              with open('./2010/had_checked.txt','r+') as f:
                  for i in f.readlines():
                      if i.replace('\n', '')==url:
                          info = 1
                          continue
              if info==0:
                  get_right_url(url, data_info_dict)
              else:
                  continue

'''匹配正确的电影的演员页面'''


    def get_right_url(url, data_info_dict):


        driver = webdriver.PhantomJS(executable_path='./phantomjs')
        driver.get(url)
        character = driver.page_source
        html_1 = etree.HTML(character)
        url_all = html_1.xpath('//div[@id="downRegion"]/div[@class="main"]/ul[@id="moreRegion"]/li[@class="clickobj"]/h3/a/@href')

        for i in range(len(url_all)):
            actor_url = url_all[i]+u'fullcredits.html'

            driver = webdriver.PhantomJS(executable_path='./phantomjs')
            driver.get(actor_url)
            info = driver.page_source
            html = etree.HTML(info)
            movie_name = html.xpath('//div[@class="db_head"]/div[@class="clearfix"]/h1/a/text()')
            movie_year = html.xpath('//div[@class="db_head"]/div[@class="clearfix"]/p/a/text()')
            # print movie_name[0]
            # print movie_year[0]

            '''判断出正确的电影页面，调用get_actors_infor(),get_worker_list()函数'''
            try:
                if movie_year[0] == data_info_dict['year']:
                        print movie_name[0]
                        print actor_url

                        data_info_dict['演员'] = get_actors_infor(actor_url)
                        data_info_dict = get_worker_list(actor_url, data_info_dict)
                        print json.dumps(data_info_dict, encoding="UTF-8", ensure_ascii=False)

                        '''将总字典写入文件'''
                        with open('./2010/data.txt','a+') as f:
                            f.write(str(data_info_dict)+'\n')

                        with open('./2010/had_checked.txt','a+') as f:
                            f.write(url + '\n')

                        break
            except:
                pass
            
            
 '''得到ｕｒｌ页面的演员列表'''
 
          def get_actors_infor(url):

              driver = webdriver.PhantomJS(executable_path='./phantomjs')
              driver.get(url)
              cinematography_info_data = driver.page_source
              html = etree.HTML(cinematography_info_data)
              '''储存演员的字典'''
              elem = {}
              for i in html.xpath('//div[@class="db_actor"]/dl/dd'):

                  actor_url= i.xpath('div[@class="actor_tit"]//h3/a/@href')
                  actors = i.xpath('div[@class="actor_tit"]//h3/a/text()')
                  # print actors[0], actor_url[0]
                  elem[actors[0].replace('\'', '\t').replace('.', '\t')] = actor_url[0]
              # print elem
              return elem
    
      def get_worker_list(url, data_info_dict):

        driver = webdriver.PhantomJS(executable_path='./phantomjs')
        driver.get(url)
        cinematography_info_data = driver.page_source
        html = etree.HTML(cinematography_info_data)
        for i in html.xpath('//div[@class="credits_r"]/div[@class="credits_list"]'):
            a = i.xpath('h4/text()')
            '''储存工作人员的字典'''
            a_list = {}
            if a[0] =='导演 Director':
                for e in i.xpath('//h3[@class="px16 normal"]/a'):
                    director = e.xpath('text()')
                    director_url = e.xpath('@href')
                    a_list[director[0]] = director_url[0]
                try:
                    for e in i.xpath('p/a'):
                        director = e.xpath('text()')
                        director_url = e.xpath('@href')
                        a_list[director[0]] = director_url[0]
                except:
                    pass
            else:
                for e in i.xpath('p/a'):
                    worker = e.xpath('text()')
                    worker_url = e.xpath('@href')
                    a_list[worker[0]]= worker_url[0]
            '''解决字典里中文变成unicode编码的问题'''
            a_list_chinese =  json.dumps(a_list, encoding="UTF-8", ensure_ascii=False)
            a_list_chinese = str(a_list_chinese).replace('\"', "\'")

            # b = a[0].replace(' ', '7').split('7')[0] + ': ' + str(a_list_chinese)
            data_info_dict[a[0].replace(' ', '7').split('7')[0]] = str(a_list_chinese)
        return  data_info_dict




      get_find_url()





存入数据库

    # -*- coding: utf-8 -*-
    import sys
    reload(sys)
    sys.setdefaultencoding('utf-8')
    from pymongo import MongoClient




    def save_data_to_mongodb():

        '''读取文件中的数据'''
        lines = open('./2011/data.txt', 'r').readlines()
        a = 1
        for line in lines:
            line = line.replace('\n','')
            '''将数据从字符串转成字典'''
            line = eval(line)
            # print a
            # a += 1
            # print line['_id']

            '''数据导入mongodb'''

            conn = MongoClient('192.168.235.55', 27017)
            db = conn['用户名']
            db.authenticate("用户名", "密码")
            db = conn['team_behind_sc']
            table = db['Filmmaker_page']
            table.insert(line)
