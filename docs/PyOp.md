# Python Operator 
To facilitate Python coders on model developing, onnxruntime provides a way to invoke operators implemented in Python.

## Implemenation
The feature is implemented under onnxruntime/core/language_interop_ops.
All Python C API dependent code are compiled into a dynamic linked library named pywrapper.
Before calling into Python script, pywrapper will convert onnxruntime tensor(s) to numpy(s), which get converted back when done.
<p>Here is a chart illustrating the calling sequence:
<pre>
onnxruntime                          pywrapper                          script
     |                                  |                                 |
     | ------------------------------>  |                                 |
     |       call with tensor(s)        | ------------------------------> |
     |                                  |         call with numpy(s)      | 
     |                                  |                                 | compute
     |                                  | <------------------------------ |
     | <------------------------------  |           return numpys(s)      |
     |         return tensor(s)         |                                 |
</pre>

## Usage
Step 1, build onnxruntime with“--config Release --enable_language_interop_ops --build_shared_lib” and override existing onnxruntime binary with the latest, then copy onnxruntime_pywrapper.dll or libonnxruntime_pywrapper.so or libonnxruntime_pywrapper.dylib to the path where onnxruntime binary is placed.
Note:
* It is suggested to compile within the Python environment where inferencing will happen. For example, if inferencing will happen in a conda env named myconda1, please compile the binary within that environment as well;
* If "--numpy_version=..." is specified, Python operator will build with that version.

Step 2, create an onnx model containing Python operator nodes:
```python
ad1_node = helper.make_node('Add', ['A','B'], ['S'])
mul_node = helper.make_node('Mul', ['C','D'], ['P'])
py1_node = helper.make_node(op_type = 'PyOp', #required, must be 'PyOp'
                            inputs = ['S','P'], #required
                            outputs = ['L','M','N'], #required
                            domain = 'pyopmulti_1', #required, must be unique
                            input_types = [TensorProto.FLOAT, TensorProto.FLOAT], #required
                            output_types = [TensorProto.FLOAT, TensorProto.FLOAT, TensorProto.FLOAT], #required
                            module = 'mymodule', #required
                            class_name = 'Multi_1', #required
                            compute = 'compute', #optional, 'compute' by default
                            W1 = '5', W2 = '7', W3 = '9') #optional, must all be strings
ad2_node = helper.make_node('Add', ['L','M'], ['H'])
py2_node = helper.make_node('PyOp',['H','N','E'],['O','W'], domain = 'pyopmulti_2',
                            input_types = [TensorProto.FLOAT, TensorProto.FLOAT, TensorProto.FLOAT],
                            output_types = [TensorProto.FLOAT, TensorProto.FLOAT],
                            module = 'mymodule', class_name = 'Multi_2')
sub_node = helper.make_node('Sub', ['O','W'], ['F'])
graph = helper.make_graph([ad1_node,mul_node,py1_node,ad2_node,py2_node,sub_node], 'multi_pyop_graph', [A,B,C,D,E], [F])
model = helper.make_model(graph, producer_name = 'pyop_model')
onnx.save(model, './model.onnx')
```
Step 3, implement mymodule.py:
```python
class Multi_1:
    def __init__(self, W1, W2, W3):
        self.W1 = int(W1)
        self.W2 = int(W2)
        self.W3 = int(W3)
    def compute(self, S, P):
        ret = S + P
        return ret + self.W1, ret + self.W2, ret + self.W3
class Multi_2:
    def compute(self, H, N, E):
        r1, r2 = H + N, N + E
        return r1, r2
```
Step 4, copy mymodule.py into Python sys.path, then reference with onnxruntime. On Windows, please set PYTHONHOME beforehand. It should point to directory where the python is installed, such as C:\Python37 or C:\ProgramData\Anaconda3\envs\myconda1 if it is in conda.

## Supported Data Types
* TensorProto.BOOL,
* TensorProto.UINT8,
* TensorProto.UINT16,
* TensorProto.UINT32,
* TensorProto.INT16,
* TensorProto.INT32,
* TensorProto.FLOAT,
* TensorProto.DOUBLE

## Limitations
* On Windows,  "--config Debug" has known issues,  build with "--config RelWithDebInfo" if need debugging symbols;
* Due to python C API restrictions, multi-threading is disabled, meaning Python operators will run sequentially.

## Test
The operator has been tested on multiple platforms, with or without conda:

Platform | Python 3.5 | Python 3.6 | Python 3.7
----------- | ------------| -----------  | -----------
Windows | (conda) passed | (conda) passed | passed
Linux | (conda) passed | (conda) passed | passed
Mac |  (conda) passed | (conda) passed | (conda) passed

## Example
Developers could resort to PyOp during model conversion for missing operators:
```python
import os
import numpy as np
from onnx import *
from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType
from skl2onnx.common.utils import check_input_and_output_numbers

X = np.array([[1, 1], [2, 1], [3, 1.2], [4, 1], [5, 0.8], [6, 1]],dtype=np.single)
nmf = NMF(n_components=2, init='random', random_state=0)
W = np.array(nmf.fit_transform(X), dtype=np.single)

def calculate_sklearn_nmf_output_shapes(operator):
    check_input_and_output_numbers(operator, output_count_range=1, input_count_range=1)
    operator.outputs[0].type.shape = operator.inputs[0].type.shape

def convert_nmf(scope, operator, container):
    ws = [str(w) for w in W.flatten()]
    attrs = {'W':'|'.join(ws)}
    container.add_node(op_type='PyOp', name='nmf', inputs=['X'], outputs=['variable'],
                       op_version=10, op_domain='MyDomain', module='mymodule', class_name='MyNmf',
                       input_types=[TensorProto.FLOAT], output_types=[TensorProto.FLOAT], **attrs)

custom_shape_calculators = {type(nmf): calculate_sklearn_nmf_output_shapes}
custom_conversion_functions = {type(nmf): convert_nmf}
initial_types = [('X', FloatTensorType([6,2]))]
onx = convert_sklearn(nmf, '', initial_types, '', None, custom_conversion_functions, custom_shape_calculators)
with th open("model.onnx", "wb") as f:
    f.write(onx.SerializeToString())
```
mymodule.py:
```python
import numpy as np
class MyNmf:
    def __init__(self,W):
        A = []
        for w in W.split('|'):
            A.append(float(w))
        self.__W = np.array(A,dtype=np.single).reshape(6,2)
    def compute(self,X):
        return self.__W
```
