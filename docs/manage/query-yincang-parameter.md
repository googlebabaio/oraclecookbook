查看隐藏参数
```
SELECT
  a.ksppinm  "Parameter",
  decode(p.isses_modifiable,'FALSE',NULL,NULL,NULL,b.ksppstvl) "Session",
  c.ksppstvl "Instance",
  decode(p.isses_modifiable,'FALSE','F','TRUE','T') "S",
  decode(p.issys_modifiable,'FALSE','F','TRUE','T','IMMEDIATE','I','DEFERRED','D') "I",
  decode(p.isdefault,'FALSE','F','TRUE','T') "D",
  a.ksppdesc "Description"
FROM x$ksppi a, x$ksppcv b, x$ksppsv c, v$parameter p
WHERE a.indx = b.indx AND a.indx = c.indx
  AND p.name(+) = a.ksppinm
  AND UPPER(a.ksppinm) LIKE UPPER('_MV_REFRESH_USE_STATS')
ORDER BY a.ksppinm;
```
