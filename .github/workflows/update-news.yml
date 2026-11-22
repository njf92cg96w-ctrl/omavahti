#!/usr/bin/env python3
"""OmaVahti RSS fetcher: standard library only, runs on GitHub Actions."""
import concurrent.futures, datetime, email.utils, json, pathlib, re, urllib.parse, urllib.request, xml.etree.ElementTree as ET

ROOT = pathlib.Path(__file__).resolve().parent

def google(q, lang='fi'):
    p = {'q': q, 'hl': 'fi' if lang == 'fi' else 'en-US', 'gl': 'FI' if lang == 'fi' else 'US', 'ceid': 'FI:fi' if lang == 'fi' else 'US:en'}
    return 'https://news.google.com/rss/search?' + urllib.parse.urlencode(p)

FEEDS = [
 ('Suomi ja maailma','🌍','https://yle.fi/rss/uutiset/paauutiset','Yle'),
 ('Suomi ja maailma','🌍','https://yle.fi/rss/t/18-34953/fi','Yle'),
 ('Tampere','🏙️','https://yle.fi/rss/t/18-146831/fi','Yle'),
 ('Päätöksenteko','🏛️','https://www.tampere.fi/ajankohtaista/uutiset.xml','Tampereen kaupunki'),
 ('Urheilu','🏒','https://yle.fi/rss/urheilu','Yle'),
 ('Tekoäly','🤖',google('tekoäly OR "artificial intelligence" when:7d'),'Google Uutiset'),
 ('Tekoäly','🤖',google('AI technology OR OpenAI OR machine learning when:7d','en'),'Google News'),
 ('Tekoäly','🤖','https://rss.arxiv.org/rss/cs.AI','arXiv'),
 ('Liikunta-ala','🏟️',google('liikuntapalvelut OR liikuntapaikat OR liikuntapolitiikka when:14d'),'Google Uutiset'),
 ('Liikunta-ala','🏟️',google('urheilupaikat uimahallit jäähallit liikuntarakentaminen when:30d'),'Google Uutiset'),
 ('Liikunta-ala','🏟️',google('sports facilities recreation management municipal sports when:30d','en'),'Google News'),
 ('Tutkimus','🔬',google('"physical activity" research OR "exercise science" study when:30d','en'),'Google News'),
 ('Tutkimus','🔬',google('liikuntatutkimus OR fyysinen aktiivisuus tutkimus when:30d'),'Google Uutiset'),
 ('Tutkimus','🔬',google('exercise physical activity systematic review study when:30d','en'),'Google News'),
 ('Tutkimus','🔬','https://rss.arxiv.org/rss/q-bio.MN','arXiv'),
]

def clean(s):
    return re.sub(r'\s+', ' ', re.sub(r'<[^>]*>', ' ', s or '')).strip()

def txt(elem, name):
    for child in elem:
        if child.tag.rsplit('}',1)[-1] == name:
            return ''.join(child.itertext()).strip()
    return ''

def normalize_date(value):
    if not value: return ''
    try: return email.utils.parsedate_to_datetime(value).isoformat()
    except Exception: return value

def fetch(feed):
    cat, icon, url, source = feed
    req = urllib.request.Request(url, headers={'User-Agent':'Mozilla/5.0 (compatible; OmaVahti/1.5; RSS reader)', 'Accept':'application/rss+xml, application/atom+xml, application/xml, text/xml, */*'})
    try:
        with urllib.request.urlopen(req, timeout=16) as response:
            body = response.read(3_000_000)
        root = ET.fromstring(body)
        nodes = [e for e in root.iter() if e.tag.rsplit('}',1)[-1] in ('item','entry')]
        output = []
        for item in nodes[:35]:
            title = clean(txt(item,'title'))
            link = txt(item,'link')
            if not link:
                for child in item:
                    if child.tag.rsplit('}',1)[-1]=='link' and child.attrib.get('href'):
                        link=child.attrib['href'];break
            desc=clean(txt(item,'description') or txt(item,'summary') or txt(item,'content'))
            date=normalize_date(txt(item,'pubDate') or txt(item,'published') or txt(item,'updated'))
            actual=cat
            if cat=='Päätöksenteko' and not re.search('päät|lautak|valtuust|kaupunginhallit|talousarvio|investoin|liikunt|rakentam|suunnitel',title+' '+desc,re.I): actual='Tampere'
            if cat=='Tampere' and not re.search('tampere|pirkanmaa|nokian|ylöjärv|kangasala|lempäälä',title+' '+desc,re.I): continue
            if title and link and link.startswith(('https://','http://')):
                output.append(dict(id=link,cat=actual,icon=icon,title=title,desc=(desc[:210]+'…' if len(desc)>210 else desc) or 'Avaa alkuperäinen julkaisu lukeaksesi lisää.',date=date,source=source,url=link))
        return {'name':source,'category':cat,'ok':bool(output),'count':len(output),'reason':'' if output else 'Syötteessä ei ollut uutisia','items':output}
    except Exception as e:
        return {'name':source,'category':cat,'ok':False,'count':0,'reason':str(e)[:140],'items':[]}

def main():
    with concurrent.futures.ThreadPoolExecutor(max_workers=6) as pool: results=list(pool.map(fetch,FEEDS))
    seen=set();items=[]
    for r in results:
        for x in r.pop('items'):
            key=x['url'].split('?')[0]
            if key not in seen: seen.add(key);items.append(x)
    items.sort(key=lambda x:x['date'],reverse=True)
    path=ROOT/'news.json'
    if not items and path.exists():
        print('No new stories; retaining previous news.json')
        return
    payload={'updated':datetime.datetime.now(datetime.timezone.utc).isoformat(),'items':items[:250],'sources':results}
    path.write_text(json.dumps(payload,ensure_ascii=False,indent=2)+'\n',encoding='utf-8')
    print(f'Saved {len(payload["items"])} articles from {sum(r["ok"] for r in results)}/{len(results)} sources')

if __name__=='__main__': main()
