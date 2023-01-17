# Plugin-Falha-de-plantio
Plugin utilizado para mapeamento de falha de plantio na cultura de cana-de-açúcar com a utilização de imagens de drones

__revision__ = '$Format:%H$'
from qgis.core import QgsProcessing
from qgis.core import QgsProcessingAlgorithm
from qgis.core import QgsProcessingMultiStepFeedback
from qgis.core import QgsProcessingParameterRasterLayer
from qgis.core import QgsProcessingParameterVectorLayer
from qgis.core import QgsProcessingParameterFeatureSink
import processing
from qgis.PyQt.QtGui import QIcon



class FalhaDePlantioAlgorithm(QgsProcessingAlgorithm):

    def initAlgorithm(self, config=None):
        self.addParameter(QgsProcessingParameterRasterLayer('gligreenleafindex', 'Raster', defaultValue=None))
        self.addParameter(QgsProcessingParameterVectorLayer('linhaplantio', 'Linha de plantio', types=[QgsProcessing.TypeVectorLine], defaultValue=None))
        self.addParameter(QgsProcessingParameterVectorLayer('quadra', 'Contorno', types=[QgsProcessing.TypeVectorPolygon], defaultValue=None))
        self.addParameter(QgsProcessingParameterFeatureSink('FalhaDePlantio', 'Falha de plantio', type=QgsProcessing.TypeVectorAnyGeometry, createByDefault=True, supportsAppend=True, defaultValue=None))
    
    
    def processAlgorithm(self, parameters, context, model_feedback):
        # Use a multi-step feedback, so that individual child algorithm progress reports are adjusted for the
        # overall progress through the model
        feedback = QgsProcessingMultiStepFeedback(13, model_feedback)
        results = {}
        outputs = {}

        parameters['FalhaDePlantio'].destinationName = 'Falha de plantio'

        # Buffer
        alg_params = {
            'DISSOLVE': False,
            'DISTANCE': 1,
            'END_CAP_STYLE': 0,  # Arredondado
            'INPUT': parameters['quadra'],
            'JOIN_STYLE': 0,  # Arredondado
            'MITER_LIMIT': 2,
            'SEGMENTS': 5,
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['Buffer'] = processing.run('native:buffer', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(1)
        if feedback.isCanceled():
            return {}

        # Recortar o vetor pela camada de máscara
        alg_params = {
            'INPUT': parameters['linhaplantio'],
            'MASK': outputs['Buffer']['OUTPUT'],
            'OPTIONS': '',
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['RecortarOVetorPelaCamadaDeMscara'] = processing.run('gdal:clipvectorbypolygon', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(2)
        if feedback.isCanceled():
            return {}

        # Recortar raster RBG
        alg_params = {
            'ALPHA_BAND': False,
            'CROP_TO_CUTLINE': True,
            'DATA_TYPE': 0,  # Use Camada de entrada Tipo Dado
            'EXTRA': '',
            'INPUT': parameters['gligreenleafindex'],
            'KEEP_RESOLUTION': False,
            'MASK': outputs['Buffer']['OUTPUT'],
            'MULTITHREADING': False,
            'NODATA': -1,
            'OPTIONS': '',
            'SET_RESOLUTION': False,
            'SOURCE_CRS': 'ProjectCrs',
            'TARGET_CRS': None,
            'X_RESOLUTION': None,
            'Y_RESOLUTION': None,
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['RecortarRasterRbg'] = processing.run('gdal:cliprasterbymasklayer', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(3)
        if feedback.isCanceled():
            return {}

        # Aritmética de bandas
        # Calculo de indeci
        alg_params = {
            'ALPHA': True,
            'FORMULA': '(2*b2 - b1 - b3)/(2*b2 + b1 + b3)',
            'INPUT': outputs['RecortarRasterRbg']['OUTPUT'],
            'OPEN': False,
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['AritmticaDeBandas'] = processing.run('lftools:bandarithmetic', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(4)
        if feedback.isCanceled():
            return {}

        # Calculadora raster
        # Calculo de pix max e pix min
        alg_params = {
            'BAND_A': 1,
            'BAND_B': None,
            'BAND_C': None,
            'BAND_D': None,
            'BAND_E': None,
            'BAND_F': None,
            'EXTRA': '',
            'FORMULA': '(A <= 0) *0 +  (A > 0) * 1',
            'INPUT_A': outputs['AritmticaDeBandas']['OUTPUT'],
            'INPUT_B': None,
            'INPUT_C': None,
            'INPUT_D': None,
            'INPUT_E': None,
            'INPUT_F': None,
            'NO_DATA': -1,
            'OPTIONS': '',
            'RTYPE': 5,  # Float32
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['CalculadoraRaster'] = processing.run('gdal:rastercalculator', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(5)
        if feedback.isCanceled():
            return {}

        # r.to.vect
        alg_params = {
            '-b': False,
            '-s': False,
            '-t': False,
            '-v': False,
            '-z': False,
            'GRASS_OUTPUT_TYPE_PARAMETER': 0,  # auto
            'GRASS_REGION_CELLSIZE_PARAMETER': 0,
            'GRASS_REGION_PARAMETER': None,
            'GRASS_VECTOR_DSCO': '',
            'GRASS_VECTOR_EXPORT_NOCAT': False,
            'GRASS_VECTOR_LCO': '',
            'column': 'DN',
            'input': outputs['CalculadoraRaster']['OUTPUT'],
            'type': 2,  # area
            'output': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['Rtovect'] = processing.run('grass7:r.to.vect', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(6)
        if feedback.isCanceled():
            return {}

        # Extrair por atributo
        alg_params = {
            'FIELD': 'DN',
            'INPUT': outputs['Rtovect']['output'],
            'OPERATOR': 0,  # =
            'VALUE': '0',
            'FAIL_OUTPUT': QgsProcessing.TEMPORARY_OUTPUT,
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['ExtrairPorAtributo'] = processing.run('native:extractbyattribute', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(7)
        if feedback.isCanceled():
            return {}

        # Excluir buracos
        alg_params = {
            'INPUT': outputs['ExtrairPorAtributo']['OUTPUT'],
            'MIN_AREA': 0.05,
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['ExcluirBuracos'] = processing.run('native:deleteholes', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(8)
        if feedback.isCanceled():
            return {}

        # Suavização
        alg_params = {
            'INPUT': outputs['ExtrairPorAtributo']['FAIL_OUTPUT'],
            'ITERATIONS': 1,
            'MAX_ANGLE': 180,
            'OFFSET': 0.25,
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['Suavizao'] = processing.run('native:smoothgeometry', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(9)
        if feedback.isCanceled():
            return {}

        # Simplificar
        alg_params = {
            'INPUT': outputs['ExtrairPorAtributo']['FAIL_OUTPUT'],
            'METHOD': 0,  # Distância (Douglas-Peucker)
            'TOLERANCE': 0.05,
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['Simplificar'] = processing.run('native:simplifygeometries', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(10)
        if feedback.isCanceled():
            return {}

        # Diferença
        alg_params = {
            'INPUT': outputs['RecortarOVetorPelaCamadaDeMscara']['OUTPUT'],
            'OVERLAY': outputs['Simplificar']['OUTPUT'],
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['Diferena'] = processing.run('native:difference', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(11)
        if feedback.isCanceled():
            return {}

        # Multipartes para partes simples
        alg_params = {
            'INPUT': outputs['Diferena']['OUTPUT'],
            'OUTPUT': QgsProcessing.TEMPORARY_OUTPUT
        }
        outputs['MultipartesParaPartesSimples'] = processing.run('native:multiparttosingleparts', alg_params, context=context, feedback=feedback, is_child_algorithm=True)

        feedback.setCurrentStep(12)
        if feedback.isCanceled():
            return {}

        # Calc_Linha_falha
        alg_params = {
            'FIELD_LENGTH': 20,
            'FIELD_NAME': 'Comprimento',
            'FIELD_PRECISION': 2,
            'FIELD_TYPE': 0,  # Float
            'FORMULA': ' $length ',
            'INPUT': outputs['MultipartesParaPartesSimples']['OUTPUT'],
            'OUTPUT': parameters['FalhaDePlantio']
        }
        outputs['Calc_linha_falha'] = processing.run('native:fieldcalculator', alg_params, context=context, feedback=feedback, is_child_algorithm=True)
        results['FalhaDePlantio'] = outputs['Calc_linha_falha']['OUTPUT']
        return results

    def name(self):
        return 'Falha de plantio'

    def displayName(self):
        return 'Falha de plantio'

    def group(self):
        return 'Análise'

    def groupId(self):
        return 'Análise'

    def icon (self):
        return QIcon(r'''C:\Users\Usuario-ASUS\Desktop\Automaçao\UBV\MODELOS\Script\Logo\martinho.png''')
        

    def createInstance(self):
        return FalhaDePlantioAlgorithm()
