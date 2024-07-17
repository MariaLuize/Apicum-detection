# Detecção de áreas de apicum em dados Landsat utilizando redes neurais profundas

Apicuns, denominadas em inglês de _hypersaline tidal flats_, são unidades geoambientais comumente associadas a presença de manguezais em áridas e semiáridas regiões ao longo da Zona Costeira Brasileira (ZCB). Tais áreas apresentam altos níveis de salinidade aos quais advém da exposição a ciclos consecutivos de inundação e seca. Estes realizam manutenção da biodiversidade local e fomento social e econômico para a comunidade local, além de agir como guarda e _buffer_ de florestas de mangue diante de aumentos no nível do mar. Entretanto, tais áreas estão à mercê de impactos ambientais, fenômenos naturais e antropológicos, e estudos focados nesse tipo de classe de uso e cobertura ainda são escassos e esparsos, logo, sua identificação e monitoramento ao longo de extensivas séries temporais se mostram tarefas importante para a sua preservação bem como forma de avaliar e descrever os impactos de atividades humanas e das mudanças climáticas, juntamente com a situação ecológica, financeira e social do espaço costeiro. Nacionalmente, o [Projeto MapBiomas](https://brasil.mapbiomas.org) é uma referência notável no que tange a classificação e detecção, ao nível de _pixel_, de alvos ambientais em longas séries de imagens de satélite. Estudos deste projeto foram relatados em publicação recente a aplicação de modelos de _Random Forest_ para a classificação de apicum em específico. Nesse sentido, por meio da execução de um mapeamento sistemático da literatura, verificou-se uma proeminência e popularidade na utilização de técnicas de _Convolucional Neural Network_ para a segmentação semântica de alvos costeiros em imagens de satélite, em especial, redes derivadas do famoso modelo U-Net, utilizando técnicas de correção atmosférica para pré-processamento e métricas de acurácia global como forma de avaliação dos mapas e dados gerados. Destarte, o presente trabalho propõe a utilização de uma rede neural convolucional do tipo U-Net para sistematicamente classificar, ao nível de _pixel_, áreas de apicum presentes em imagens de satélite da missão Landsat. Tais imagens a serem classificadas compreendem uma série temporal com início em 1985 e término em 2023, embargando a ZCB. Ainda, técnicas de pré e pós-processamentos foram utilizada.


## Google EARTH Engine Scripts
* Geração de mosaicos: [link](https://code.earthengine.google.com/539fc0c39830cb4525256e91bf0e88a4)

* Função de preenchimento e filtro temporal: [link](https://code.earthengine.google.com/f86fb59789dcdfc6f57bd3af42c02ecc)
* Filtro espacial: [link](https://code.earthengine.google.com/09b02da1105ed2b5703921fd9b69a480)
* Filtro de frequência: [link](https://code.earthengine.google.com/7f52208abe29145d3ea9ba847fd5dba7)
* Delimitação por área da ZCB: [link](https://code.earthengine.google.com/78f506affaadf361f203b2480568de0f)

* Visualização do produto final e estatísticas: [link](https://code.earthengine.google.com/335420df1616c85acf67f99bcc951a88)*


## Uso do script de ''Visualização do produto final e estatísticas''
* NECESSARY IMPORTS
```javascript
var geometry_RJ = 
    /* color: #d63000 */
    /* displayProperties: [
      {
        "type": "rectangle"
      }
    ] */
    ee.Geometry.Polygon(
        [[[-44.41376933593751, -22.106248427099384],
          [-44.41376933593751, -23.55919889599254],
          [-42.75483378906251, -23.55919889599254],
          [-42.75483378906251, -22.106248427099384]]], null, false),
    imageVisParam = {"opacity":1,"bands":["swir1","nir","red"],"min":1,"max":104,"gamma":1};
```
* BASIC CONFIGURATIONS
```javascript
// YEAR SELECTION
var year    = 1985;
var classID = 32;
var mosaic  = ee.Image("projects/mapbiomas-workspace/TRANSVERSAIS/ZONACOSTEIRA6/mosaic_"+year);

// BASIC MOSAIC VISUALIZATION
Map.addLayer(mosaic,imageVisParam,'Mosaic '+year);

// BLANK CANVAS FOR BETTER VISUALIZATION
Map.addLayer(ee.Image(0),{palette:'FFFFFF'},'Blank',false);
```

* VISUALIZATION OF MapBiomas 6 CLASSIFICATION
```javascript
// ATTENTION: PLEASE, DO NOT MODIFY THE FOLLOWING LINES
var merge = ee.ImageCollection('projects/mapbiomas-workspace/TRANSVERSAIS/ZONACOSTEIRA6-FT')
             .filterMetadata('version','equals','3').filter(ee.Filter.eq('year',year)).max().eq(classID).unmask(0);
var displacedMergeMask = merge.focal_max(4).reproject('EPSG:4326', null, 30)
merge = merge.updateMask(displacedMergeMask.eq(1)).unmask(0)
Map.addLayer(merge,{palette:['white','red'],min:0,max:1},'Reference Mapbiomas Collection 6',false)
```

* SORT POINTS
```javascript
var points = merge.stratifiedSample(10000,'classification',geometry_RJ,30,null,1,[0,1],[5000,5000],true,1,true)
print(points.size())
print(points.first())
Map.addLayer(points,{},'Points',false)
```
* HYPERSALINE TIDAL FLATS (HTF) CLASSIFICATION
```javascript
// ATTENTION: PLEASE, DO NOT MODIFY THE FOLLOWING LINES
var HTF_Seg = ee.ImageCollection('projects/masters-exp-postprocessing/assets/Data_Augmentation/clear_product')
                                                .filterMetadata('version','equals',14).filter(ee.Filter.eq('year',year)).max().unmask(0).rename('classification');
```
* COMPARISON BETWEEN REFERENCE MAP (MapBiomas 6) WITH HTF CLASSIFICATION
```javascript
points= points.map(function(feat){
  return feat.set({'HTF_Seg':HTF_Seg.eq(classID).reduceRegion(ee.Reducer.first(),feat.geometry(),30).get('classification')})
})
print(points.first())

Map.addLayer(HTF_Seg.mask(HTF_Seg.eq(classID)),{palette:'blue'},'HTF Segmentation',false)
```

* CORRELATIONS
```javascript
//No-change
var noChangeP = HTF_Seg.eq(classID).and(merge.eq(1))
var noChangeN = HTF_Seg.neq(classID).and(merge.eq(0))

//Positive-Disagreement (HTF_Seg HTF -> ecosystem non-HTF)
var positiveDisagreement = HTF_Seg.eq(classID).and(merge.neq(1))

//Negative-Disagreement (HTF_Seg non-HTF -> ecosystem HTF)
var nagativeDisagreement = HTF_Seg.neq(classID).and(merge.eq(1))

var changes = ee.ImageCollection([ee.Image(1).toByte().mask(noChangeP),
                                  ee.Image(2).toByte().mask(positiveDisagreement),
                                  ee.Image(3).toByte().mask(nagativeDisagreement)
                                  ]).max().unmask(0)
Map.addLayer(changes.mask(changes.gt(0)),{palette:['000000','0000FF','00FF00','FF0000'],min:0,max:3},'Changes')
```

* STATISTICS
```javascript
var statesBR = ee.FeatureCollection('projects/solvedltda/assets/BR_UF_2021')
 var changesPerState = statesBR.map(function(feat){
      var nochangesP = noChangeP.reduceRegion({reducer:ee.Reducer.sum(),geometry:feat.geometry(),scale:30,maxPixels:1e13})
      var nochangesN = noChangeN.reduceRegion({reducer:ee.Reducer.sum(),geometry:feat.geometry(),scale:30,maxPixels:1e13})
      var positive = positiveDisagreement.reduceRegion({reducer:ee.Reducer.sum(),geometry:feat.geometry(),scale:30,maxPixels:1e13})
      var negative = nagativeDisagreement.reduceRegion({reducer:ee.Reducer.sum(),geometry:feat.geometry(),scale:30,maxPixels:1e13})
      var sigla = feat.get('sigla')
    return ee.Feature(null,{'nochangeN':nochangesN.get('classification'),'nochangeP':nochangesP.get('classification'),'positive':positive.get('classification'),'negative':negative.get('classification'),'sigla':sigla})
 })
var changesPerState_Exemple = changesPerState.first()
print('Brasilian state exemple: ', changesPerState_Exemple.get('sigla'), changesPerState_Exemple) 
Export.table.toDrive(changesPerState,'changes_per_state_'+year+'_mapbiomas','results_HTF','changes_per_state_'+year+'_mapbiomas')
```

* LOCALIZATION SET
```javascript
Map.setCenter(-43.59631,-23.01244,13)
```

* POINTS BASED STATS
```javascript
var errorM = points.errorMatrix('classification','HTF_Seg')
print('Error Matrix',errorM)
print('O.A',errorM.accuracy())
print('F1-Score',errorM.fscore())
print('Consumer',errorM.consumersAccuracy())
print('Producers',errorM.producersAccuracy())
