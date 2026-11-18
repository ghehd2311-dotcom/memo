영문 [출처](https://nat-arslan.github.io/posts/Obsidian-PostgreSQL-Python/#:~:text=From%20there%20on%20I%20can%20connect%20to,in%20Python%20to%20analyse%20and%20visualise%20notes.)


현재 화면에 공유되어 있는 **"Using Obsidian, PostgreSQL and Python to Analyse & Visualise Notes"** 게시글의 주요 내용을 한국어로 번역 및 요약해 드립니다.

  

## 📝 옵시디언(Obsidian), PostgreSQL, 파이썬을 활용한 노트 분석 및 시각화

**작성자:** [Nat Arslan](https://nat-arslan.github.io/posts/Obsidian-PostgreSQL-Python/#:~:text=From%20there%20on%20I%20can%20connect%20to,in%20Python%20to%20analyse%20visualise%20notes) (게시일: 2022년 10월 30일, 수정일: 2022년 11월 1일)


  

저는 [Obsidian](https://nat-arslan.github.io/posts/Obsidian-PostgreSQL-Python/#:~:text=From%20there%20on%20I%20can%20connect%20to,in%20Python%20to%20analyse%20visualise%20notes)을 사용하여 마크다운 형식으로 노트와 일지를 작성합니다. 이 앱을 통해 서로 연관된 노트를 만들고, 관계를 그래프화하며, 태그나 수십 개의 플러그인을 활용해 노트를 이해할 수 있습니다. 하지만 항상 더 많은 패턴과 연결고리를 보고 싶다는 갈증이 있었고, 이를 달성하는 데 파이썬(Python)만한 도구는 없습니다.

  

각 파일의 메타데이터(YAML)를 [PostgreSQL](https://nat-arslan.github.io/posts/Obsidian-PostgreSQL-Python/#:~:text=From%20there%20on%20I%20can%20connect%20to,in%20Python%20to%20analyse%20visualise%20notes) 데이터베이스로 쉽게 전송할 수 있는 옵시디언 [플러그인](https://nat-arslan.github.io/posts/Obsidian-PostgreSQL-Python/#:~:text=From%20there%20on%20I%20can%20connect%20to,in%20Python%20to%20analyse%20visualise%20notes)이 존재합니다. 이를 활용하면 데이터베이스에 접속하여 파이썬으로 일지 항목들을 분석할 수 있습니다.

  

### 🔄 전체 작업 흐름 (Workflow)

1. **옵시디언에서 꾸준히 노트 작성하기**
    
      
    - 일관된 메타데이터(YAML) 섹션을 유지하는 것이 중요합니다. 추적/기록할 수 있는 항목의 예시로는 **날짜, 기분, 날씨, 위치** 등이 있습니다.
        
          
        
2. **옵시디언용 PostgreSQL 플러그인 다운로드**
    
      
    - 노트(일지, 회의록 등)를 작성한 후 이 플러그인을 사용하여 파일의 메타데이터를 PostgreSQL 데이터베이스로 전송합니다.
        
          
        
3. **파이썬에서 SQL을 사용하여 노트 분석 및 시각화하기** (자세한 단계는 [깃허브 저장소](https://nat-arslan.github.io/posts/Obsidian-PostgreSQL-Python/#:~:text=From%20there%20on%20I%20can%20connect%20to,in%20Python%20to%20analyse%20visualise%20notes) 참조)
    
      
    

### 💻 파이썬에서의 주요 분석 단계 요약

1. **주피터 노트북(Jupyter Notebook)에서 PostgreSQL 데이터베이스 연결**
    
      
    
    Python
    
    ```
    %sql postgresql://username:password@host:port/database
    ```
    
2. **SQL 쿼리 작성**
    
      
    
    Python
    
    ```
    query = ''' 
        SELECT dataview_data -> 'date' as date, 
               dataview_data -> 'tags' as tags, 
               dataview_data -> 'mood' as mood, 
               dataview_data -> 'geom' as geom 
        FROM obsidian.file; 
    '''
    ```
    
3. **SQL 쿼리와 SQLAlchemy 엔진을 사용해 DB 데이터를 Pandas로 변환**
    
      
    
    Python
    
    ```
    df = pd.read_sql(query, engine)
    ```
    
4. **필요에 따라 데이터프레임 편집**
    
      
    
5. **데이터프레임을 GeoPandas로 변환하여 IpyLeaflet으로 지도 시각화**
    
      
    
6. **데이터 플로팅(시각화)**
    
      
    
7. **(선택 사항) 선호에 따라 Streamlit을 사용하여 대시보드 구축**
    
      
    

노트 간의 숨겨진 연관성을 밝히기 위해 `dataframe.corr()` 함수를 사용하여 메타데이터 필드 간의 상관관계를 확인할 수도 있습니다. 이는 _"산책을 한 날에 기분이 더 좋아질까?"_ 같은 질문에 답하는 데 도움을 줍니다.

  

손으로 쓰는 일지의 장점도 많지만, 기록한 정보를 직접 분석할 수 있어야 합니다. 데이터를 PostgreSQL로 보낸 뒤 파이썬으로 연결하는 방식은 이를 수행하기 위한 매우 효율적인 방법임이 증명되었습니다.