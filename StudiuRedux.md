In exemplul de mai jos observam un model de dashboard ce contine un 
tabel de administrare al unor produse.
Asupra acestui tabel multifunctional se aplica metodele 
{handleSelect,handleSave,handleCancel,handleModifications,handleDelete},
metode care pe langa rolul lor functional fac si un request la baza de date
pentru a face anumite operatii.

Observam ca aceasta componenta Dashboard contine foarte multe elemente ce 
fac codul greu de inteles. De asemenea crearea unui alt tabel pentru alta entitate
devine o operatie repetitiva si complicata ce nu se scaleaza usor.

Se pune problema crearii unei componente AdminTable ce reprezinta o tabela
ce poate fi modificata dar care poate face diverse requesturi pentru mai multe tipuri 
de entitati indiferent de structura acestora. Astfel vom putea avea acces la o interfata de administrare 
mult mai versatila in care tabela si structura acesteia poate fi schimbata in functie de modelul necesar.

Scop: Sa putem avea o tabela pentru Produse, Case si Adrese si sa putem schimba intre ele intr-un mod foarte facil.

#### /src/Dashboard.jsx

```javascript
import React,{useEffect, useState,useRef} from 'react';
import axios from 'axios';
import {Redirect} from 'react-router-dom';


const TableEditableRow = ({values,onSelect,onSave,onChange,onCancel,onDelete,selected}) => {

    const unusedKeys = ['id','createdAt','updatedAt'];

    const renderCellValue = (key, values) => {
        if(selected)
            return <input name={key} onChange={onChange} value={values[key]}/>
        return values[key];
    }

    return <React.Fragment>
            <tr>
                {Object.keys(values)
                .filter(key => unusedKeys.indexOf(key)<0)
                .map((key,index) => (
                    <td key={index} onClick={onSelect}> 
                        {renderCellValue(key,values)}
                    </td>
                ))}
                
                {selected && (
                    <React.Fragment>
                        <td>
                            <button onClick={onSave}>SAVE</button>
                        </td>
                        <td>
                            <button onClick={onCancel}>CANCEL</button>
                        </td>
                        <td>
                            <button onClick={onDelete}>DELETE</button>
                        </td>
                    </React.Fragment>
                )}
            </tr>
        </React.Fragment>
}

const Dashboard = () => {
    
    const [redirect,setRedirect] = useState(null);
    const [loading, setLoading] = useState(true);
    const [products, setProducts] = useState([]);
    const [formData, setFormData] = useState({});
    const [selectedProduct,setSelectedProduct] = useState({});
    const postForm = useRef();

    useEffect(()=> {
        axios.get('http://localhost:8080/login',{withCredentials:true})
        .then(response=>response.data)
        .then(({message}) => {
            setRedirect(message !== "ok");
            setLoading(false);
        })
        .catch((error) => {
            setRedirect(true);
            setLoading(false);
        });

        axios.get('http://localhost:8080/product',{withCredentials:true})
        .then(result => setProducts(result.data))
        .catch(error => console.log(error));
    },[]);

    const handleDisconnect = () => {
        axios.get('http://localhost:8080/login/disconect',{withCredentials:true})
        .then(result => {
            setRedirect(true);
        })
        .catch(error => {
            console.log(error);
            setRedirect(true);
        });
    }

    const handleFormChange = (event) => {
        setFormData({
            ...formData,
            [event.target.name]: event.target.value
            })
    }

    const handleDelete = () => {
        const {id} = selectedProduct;
        const index = products.findIndex((prod) => prod.id === id);
        const start = products.slice(0, index);
        const end = products.slice(index + 1);
        setProducts([...start, ...end]);
        axios.delete('http://localhost:8080/product',{params:{id},withCredentials:true})
        .then(() => {})
        .catch(error =>{
            console.error(error);
            setRedirect(true);
        })
        setSelectedProduct({});
    }

    const handleSelect = (product) => {
        if (selectedProduct.id === product.id)
            return;

        if (Object.keys(selectedProduct).length > 0) {
            const index = products.findIndex((prod) => prod.id === selectedProduct.id);
            const p = {...selectedProduct}
            const start = products.slice(0,index);
            const end = products.slice(index+1);
            setProducts([...start,p,...end]);
        }

        setSelectedProduct(product);
    }

    const handleModification = (event) => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        const p = {...products[index],[event.target.name]:event.target.value}
        const start = products.slice(0,index);
        const end = products.slice(index+1);
        setProducts([...start,p,...end]);
    }

    const handleCancel = () => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        setSelectedProduct({});
        const p = {...selectedProduct}
        const start = products.slice(0,index);
        const end = products.slice(index+1);
        setProducts([...start,p,...end]);
    }
    
    

    const handleSave = () => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        axios.put('http://localhost:8080/product',{...products[index]},{withCredentials:true})
        .then(response => response.data)
        .then((response) => {})
        .catch((error) => {
            console.error(error);
            setRedirect(true);
        });
        setSelectedProduct({});
    }

    //handle add on submit
    const handleFormSubmit = (event) => {
        event.preventDefault();
        postForm.current.reset();
    }

    const handleAddProduct = () => {
        const {name,price} = formData;

        if(name.length < 3 && price <= 0)
            return;

        axios.post('http://localhost:8080/product',formData,{withCredentials:true})
        .then(({data}) => setProducts([...products,data]))
        .catch(error =>{
            console.log(error);
            setRedirect(true);
        });
    }

    if (loading) {
        return <React.Fragment>Loading...</React.Fragment>
    }

    if (redirect) {
        return <Redirect to='/login'/>
    }

    return (<React.Fragment>
        Dashboard
        <button onClick={handleDisconnect}>Disconnect</button>
        <form
        ref={postForm}
        onChange={handleFormChange}
        onSubmit={handleFormSubmit}>
        <table>
            <thead>
                <tr>
                    <th>Name</th>
                    <th>Price</th>
                    <th>Action</th>
                </tr>
            </thead>
            <tbody>
                <tr>
                    <td>
                        <input name="name" type="text" placeholder="Product name"/>
                    </td>
                    <td>
                        <input name="price" type="number" placeholder="Price"/>
                    </td>
                    <td>
                        <button onClick={handleAddProduct}>Add Product</button>
                    </td>
                </tr>
            </tbody>
        </table>    
        </form>

        <table>
            <tbody>
                {products.map((product) => <TableEditableRow
                                        onSelect={()=>{handleSelect(product)}}
                                        onSave={handleSave}
                                        onCancel={handleCancel}
                                        onChange={handleModification}
                                        onDelete={handleDelete}
                                        selected={product.id === selectedProduct.id}
                                        values={product}
                                        key={product.id} />)}
            </tbody>
        </table>
    </React.Fragment>);
}

export default Dashboard;

```

Un prim pas in abordarea acestei implementari este curatarea codului prezent mai sus.
Observam ca un element ce se repeta frecvent si ingreuneaza lizibilitatea codului este requestul la API 
facut cu axios. Astfel optam pentru implementarea unei serii de 'Endpointuri' ce ne vor intoarce 
promisiunile acelor requesturi. Astfel obtinem:

#### /src/endpoints/index.js

```javascript
import axios from 'axios';

const HOST = 'http://localhost:8080';

const PATHS = {
    PRODUCT: '/product',
    LOGIN: '/login',
    DISCONECT: '/login/disconnect'
}

for (const key of Object.keys(PATHS)) {
    PATHS[key] = `${HOST}${PATHS[key]}`;
}

export default {
    product:{
        get: () => axios.get(PATHS.PRODUCT,{withCredentials:true}),
        post: (payload) => axios.post(PATHS.PRODUCT,{...payload},{withCredentials:true}),
        put: (payload) => axios.put(PATHS.PRODUCT,{...payload},{withCredentials:true}),
        delete: (id) => axios.delete(PATHS.PRODUCT,{params:{id},withCredentials:true})
    },
    login:{
        authenticate: () => axios.get(PATHS.LOGIN,{withCredentials:true}),
        disconnect: () => axios.get(PATHS.DISCONECT,{withCredentials:true})
    }
}
```

Prin aceasta modificare vom importa Endpoints si vom utiliza try...catch
impreuna cu async/await pentru a obtine un cod mai aerisit ce are functionalitatea
similara celui de mai sus.

#### /src/Dashboard.jsx

```javascript
import React,{useEffect, useState,useRef} from 'react';
import {Redirect} from 'react-router-dom';
import Endpoints from './endpoints';

//...TableEditableRow definition

const Dashboard = () => {
    
    const [redirect,setRedirect] = useState(null);
    const [loading, setLoading] = useState(true);
    const [products, setProducts] = useState([]);
    const [formData, setFormData] = useState({});
    const [selectedProduct,setSelectedProduct] = useState({});
    const postForm = useRef();

    useEffect(() => {
        const authenticate = async () => {
            try {
                const {data} = await Endpoints.login.authenticate();
                setRedirect(data.message !== "ok");
                setLoading(false);
            }
            catch(error) {
                setRedirect(true);
                setLoading(false);
            }
        }
    
        const getProducts = async () => {
            try {
                const {data} = await Endpoints.product.get();
                setProducts(data);
            }
            catch(error) {
                console.log(error);
            }
        }

        authenticate();
        getProducts();
    },[]);

    const handleDisconnect = async () => {
        try {
            await Endpoints.login.disconnect();
            setRedirect(true);
        }
        catch (error) {
            console.log(error);
            setRedirect(true);
        }
    }

    const handleFormChange = (event) => {
        setFormData({
            ...formData,
            [event.target.name]: event.target.value
            })
    }

    const handleDelete = async () => {
        const {id} = selectedProduct;
        const index = products.findIndex((prod) => prod.id === id);
        const start = products.slice(0, index);
        const end = products.slice(index + 1);
        setProducts([...start, ...end]);
        setSelectedProduct({});
        try {
            await Endpoints.product.delete(id);
        }
        catch (error) {
            console.error(error);
        }
        
    }

    const handleSelect = (product) => {
        if (selectedProduct.id === product.id)
            return;

        if (Object.keys(selectedProduct).length > 0) {
            const index = products.findIndex((prod) => prod.id === selectedProduct.id);
            const p = {...selectedProduct}
            const start = products.slice(0,index);
            const end = products.slice(index+1);
            setProducts([...start,p,...end]);
        }

        setSelectedProduct(product);
    }

    const handleModification = (event) => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        const p = {...products[index],[event.target.name]:event.target.value}
        const start = products.slice(0,index);
        const end = products.slice(index+1);
        setProducts([...start,p,...end]);
    }

    const handleCancel = () => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        const p = {...selectedProduct}
        const start = products.slice(0,index);
        const end = products.slice(index+1);
        setProducts([...start,p,...end]);
        setSelectedProduct({});
    }

    const handleSave = async () => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        try {
            await Endpoints.product.put(products[index]);
        }
        catch (error) {
            console.error(error);
        }
        setSelectedProduct({});
    }

    const handleFormSubmit = (event) => {
        event.preventDefault();
        postForm.current.reset();
    }

    const handleAddProduct = async () => {
        const {name,price} = formData;

        if(name.length < 3 && price <= 0)
            return;

        try {
           const {data} = await Endpoints.product.post(formData);
           setProducts([...products,data]);
        }
        catch (error) {
            console.log(error);
        }
    }

   //...login, redirect, return
}

export default Dashboard;
```

O alta observatie pe care o putem face este aceea ca repetam scrierea insertilor imutabile. In acest studiu
optam pentru adaugarea a doua functii noi la prototipul obiectului Array. Pentru ca aceste functii sa fie disponibile pe
tot parcursul proiectului alegem sa adaugam declararea acestora in index.js.

#### src/index.js

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';
import App from './App';
import * as serviceWorker from './serviceWorker';

Array.prototype.immutableRemove = function(index) {
  const start = this.slice(0, index);
  const end = this.slice(index + 1);
  return [...start, ...end];
}


Array.prototype.immutableInsertAt = function(index, value) {
  const start = this.slice(0,index);
  const end = this.slice(index+1);
  return [...start,{...value},...end];
}

ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById('root')
);

// If you want your app to work offline and load faster, you can change
// unregister() to register() below. Note this comes with some pitfalls.
// Learn more about service workers: https://bit.ly/CRA-PWA
serviceWorker.unregister();
```

Asadar putem folosi acum functiile immutableInsertAt si immutableRemove pe orice array si ne va returna 
o variabila noua, dar ce va avea o structura diferita. Aceste operatii imutabile se asigura ca apelarea
functiilor setState rerandeaza elementele in cauza. Dupa aceasta modificare Dasboard jsx arata astfel

#### /src/Dashboard.jsx

```javascript
import React,{useEffect, useState,useRef} from 'react';
import {Redirect} from 'react-router-dom';
import Endpoints from './endpoints';

//...TableEditableRow definition

const Dashboard = () => {
    
    const [redirect,setRedirect] = useState(null);
    const [loading, setLoading] = useState(true);
    const [products, setProducts] = useState([]);
    const [formData, setFormData] = useState({});
    const [selectedProduct,setSelectedProduct] = useState({});
    const postForm = useRef();

    useEffect(() => {
        const authenticate = async () => {
            try {
                const {data} = await Endpoints.login.authenticate();
                setRedirect(data.message !== "ok");
                setLoading(false);
            }
            catch(error) {
                setRedirect(true);
                setLoading(false);
            }
        }
    
        const getProducts = async () => {
            try {
                const {data} = await Endpoints.product.get();
                setProducts(data);
            }
            catch(error) {
                console.log(error);
            }
        }

        authenticate();
        getProducts();
    },[]);

    const handleDisconnect = async () => {
        try {
            await Endpoints.login.disconnect();
            setRedirect(true);
        }
        catch (error) {
            console.log(error);
            setRedirect(true);
        }
    }

    const handleFormChange = (event) => {
        setFormData({
            ...formData,
            [event.target.name]: event.target.value
            });
    }

    const handleDelete = async () => {
        const {id} = selectedProduct;
        const index = products.findIndex((prod) => prod.id === id);
        setProducts(products.immutableRemove(index));
        setSelectedProduct({});
        try {
            await Endpoints.product.delete(id);
        }
        catch (error) {
            console.error(error);
        }
        
    }

    const handleSelect = (product) => {
        if (selectedProduct.id === product.id)
            return;

        if (Object.keys(selectedProduct).length > 0) {
            const index = products.findIndex((prod) => prod.id === selectedProduct.id);
            setProducts(products.immutableInsertAt(index,selectedProduct));
        }

        setSelectedProduct(product);
    }

    const handleModification = (event) => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        const p = {...products[index],[event.target.name]:event.target.value};
        setProducts(products.immutableInsertAt(index, p));
    }

    const handleCancel = () => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        setProducts(products.immutableInsertAt(index,selectedProduct));
        setSelectedProduct({});
    }

    const handleSave = async () => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        try {
            await Endpoints.product.put(products[index]);
        }
        catch (error) {
            console.error(error);
        }
        setSelectedProduct({});
    }

    const handleFormSubmit = (event) => {
        event.preventDefault();
        postForm.current.reset();
    }

    const handleAddProduct = async () => {
        const {name,price} = formData;

        if(name.length < 3 && price <= 0)
            return;

        try {
           const {data} = await Endpoints.product.post(formData);
           setProducts([...products,data]);
        }
        catch (error) {
            console.log(error);
        }
    }
   //...login, redirect, return
}

export default Dashboard;
```

Urmatorul pas este acela de a implementa serviciul redux/react-redux. Pentru acest serviciu avem nevoie de 
3 elemente: un store, un reducator si un manager de actiuni.

In primul rand configuram un store pe care il adaugam in jurul Routerului principal cu ajutorul componentei Provider.

#### src/App.jsx

```javascript
import React from 'react';
import {Provider} from 'react-redux';
import {BrowserRouter as Router, Switch,Route} from 'react-router-dom';

import Login from './Login';
import Dashboard from './Dashboard';
import store from './redux';

const App = ()=>{
  
  return (
      <Provider store={store}>
        <Router>    
          <Switch>
            <Route path='/login' component={Login}/>
            <Route path='/dashboard' component={Dashboard}/>
          </Switch>
        </Router>
      </Provider>
      
    
  );
}

export default App;

```

#### src/redux/index.js

```javascript
import { createStore, combineReducers } from 'redux'

import productsReducer from './productsReducer';

export default createStore(combineReducers({
    productsReducer
}));
```

Pentru crearea unui sistem oarecum versatil de management al actiunilor vom avea nevoie 
de aceste doua functii de utilitate. Acestea au rolul de a elimina repetitile celor 3 stadii ale
actiunilor asincrone, anume begin,success si failure.

#### src/redux/actionUtils.js

```javascript
export const createAsyncKeys = (name) => ({
    BEGIN: `${name}_BEGIN`,
    SUCCESS: `${name}_SUCCESS`,
    FAILURE: `${name}_FAILURE`
});

export const createAsyncActions = (asyncActionName) => ({
    begin: (payload) => ({
        type: asyncActionName.BEGIN,
        payload
    }),
    success: (payload) => ({
        type: asyncActionName.SUCCESS,
        payload
    }),
    failure: (payload) => ({
        type: asyncActionName.FAILURE,
        payload
    })
});
```

Mai jos se regaseste reducatorul pentru produse. Acest reducator are 3 parti,
declararea actiunilor(nume si creator), definirea actiunilor si partea de reducator in 
care se modifica efectiv state-ul. Actiunile asincrone se vor folosi de obiectul dispatch
primit in momentul apelului din componente. Acesta va face dispatch unei serii de actiuni in functie de 
desfasurarea procesului de fetch la endpointurile definite anterior.

#### src/redux/productReducer.js

```javascript

import {createAsyncActions, createAsyncKeys} from './actionUtils';
import Endpoints from '../endpoints';

const initialState = {
    products: [],
    selectedProduct: {}
};

//DECLARAREA ACTIUNILOR

const ACTION_NAMES = {
    ADD_NEW_PRODUCT: createAsyncKeys('ADD_NEW_PRODUCT'),
    GET_PRODUCTS: createAsyncKeys('GET_PRODUCTS'),
    DELETE_PRODUCT: createAsyncKeys('DELETE_PRODUCT'),
    HANDLE_MODIFICATIONS: 'HANDLE_MODIFICATIONS',
    CANCEL_MODIFICATIONS: 'CANCEL_MODIFICATIONS',
    SELECT_PRODUCT: 'SELECT_PRODUCT',
    UPDATE_PRODUCT: createAsyncKeys('UPDATE_PRODUCT'),
    

}

const ACTION_CREATORS = {
    addNewProduct: createAsyncActions(ACTION_NAMES.ADD_NEW_PRODUCT),
    getProducts: createAsyncActions(ACTION_NAMES.GET_PRODUCTS),
    deleteProduct: createAsyncActions(ACTION_NAMES.DELETE_PRODUCT),
    handleModifications: (payload) => ({type:ACTION_NAMES.HANDLE_MODIFICATIONS,payload}),//de continuat cu payload cu date modificate
    cancelModifications: () => ({type:ACTION_NAMES.CANCEL_MODIFICATIONS}),
    selectProduct: (payload) => ({type:ACTION_NAMES.SELECT_PRODUCT, payload}),
    updateProduct: createAsyncActions(ACTION_NAMES.UPDATE_PRODUCT)
}

///DEFINIREA ACTIUNILOR
export const actions = {
    addNewProduct: async (dispatch, newProduct) => {
        const {success,failure} = ACTION_CREATORS.addNewProduct;
        try {
            const {data} = await Endpoints.product.post(newProduct);
            dispatch(success(data));
         }
         catch (error) {
             console.log(error);
             dispatch(failure());
         }
    },
    getProducts: async (dispatch) => {
        const {success,failure} = ACTION_CREATORS.getProducts;
        try {
            const {data} = await Endpoints.product.get();
            dispatch(success(data));
        }
        catch (error) {
            console.error(error);
            dispatch(failure([]));
        }
    },
    deleteProduct: async (dispatch, productId) => {
        const {success,failure} = ACTION_CREATORS.deleteProduct;
        try {
            await Endpoints.product.delete(productId);
            dispatch(success());
        }
        catch (error) {
            console.error(error);
            dispatch(failure());
        }
    },
    handleModifications: (dispatch,payload) => {dispatch(ACTION_CREATORS.handleModifications(payload))},
    cancelModifications: (dispatch) => {dispatch(ACTION_CREATORS.cancelModifications())},
    selectProduct: (dispatch,payload) => {dispatch(ACTION_CREATORS.selectProduct(payload))},
    updateProduct: async (dispatch, product) => {
        const {success,failure} = ACTION_CREATORS.updateProduct;
        try {
            await Endpoints.product.put(product);
            dispatch(success());
        }
        catch (error) {
            console.error(error);
            dispatch(failure());
        }
    }
}

///CASEURILE CE DEFINESC COMPORTAMENTUL
///STATE-ULUI IN FUNCTIE DE ACTIUNI
export default (state = initialState, action) => {
    const {type,payload} = action;
    switch(type){
        case ACTION_NAMES.ADD_NEW_PRODUCT.SUCCESS:{
            const {products} = state;
            return {
                ...state,
                products: [...products,payload]
            }
        }
        case ACTION_NAMES.GET_PRODUCTS.SUCCESS:
            return {
                ...state,
                products: [...payload]
            }
        case ACTION_NAMES.GET_PRODUCTS.FAILURE:
            return {
                ...state,
                products: [...payload]
            }
        case ACTION_NAMES.SELECT_PRODUCT:
            return {
                ...state,
                selectedProduct: {...payload} 
            }
        case ACTION_NAMES.HANDLE_MODIFICATIONS:{
            const {products, selectedProduct} = state;
            const index = products.findIndex(({id}) => id === selectedProduct.id);
            const p = {...products[index],[payload.name]:payload.value};
            // setProducts();
            return {
                ...state,
                products: products.immutableInsertAt(index, p)
            }
        }
        case ACTION_NAMES.CANCEL_MODIFICATIONS: {
            const {products, selectedProduct} = state;
            const index = products.findIndex(({id}) => id === selectedProduct.id);
            return {
                ...state,
                products: products.immutableInsertAt(index,selectedProduct),
                selectedProduct: {}
            }
        }
        case ACTION_NAMES.DELETE_PRODUCT.SUCCESS:{
            const {products, selectedProduct} = state;
            const index = products.findIndex(({id}) => id === selectedProduct.id);
            return {
                ...state,
                products: state.products.immutableRemove(index),
                selectedProduct: {}
            }
        }
        case ACTION_NAMES.DELETE_PRODUCT.FAILURE:
            return {
                ...state,
                selectedProduct: {}
            }
        case ACTION_NAMES.UPDATE_PRODUCT.SUCCESS: {
            return {
                ...state,
                selectedProduct: {}
            }
        }
        case ACTION_NAMES.UPDATE_PRODUCT.FAILURE: {
            const {products, selectedProduct} = state;
            const index = products.findIndex(({id}) => id === selectedProduct.id);
            return {
                ...state,
                products: products.immutableInsertAt(index, selectedProduct),
                selectedProduct: {}
            }
        }
        default:
            return state;
    }
}

```

Odata cu aceasta implementare putem crea cu usurinta o componenta numita AdminTable.jsx

#### AdminTable.jsx

```javascript
import React,{useEffect, useState,useRef} from 'react';
import {useDispatch, useSelector} from 'react-redux';
import {actions as productActions} from '../../redux/productsReducer';

const TableEditableRow = ({values,onSelect,onSave,onChange,onCancel,onDelete,selected}) => {

    const unusedKeys = ['id','createdAt','updatedAt'];

    const renderCellValue = (key, values) => {
        if(selected)
            return <input name={key} onChange={onChange} value={values[key]}/>
        return values[key];
    }

    return <React.Fragment>
            <tr>
                {Object.keys(values)
                .filter(key => unusedKeys.indexOf(key)<0)
                .map((key,index) => (
                    <td key={index} onClick={onSelect}> 
                        {renderCellValue(key,values)}
                    </td>
                ))}
                
                {selected && (
                    <React.Fragment>
                        <td>
                            <button onClick={onSave}>SAVE</button>
                        </td>
                        <td>
                            <button onClick={onCancel}>CANCEL</button>
                        </td>
                        <td>
                            <button onClick={onDelete}>DELETE</button>
                        </td>
                    </React.Fragment>
                )}
            </tr>
        </React.Fragment>
}

const AdminTable = () => {

    const [formData, setFormData] = useState({});
    const postForm = useRef();
    const dispatch = useDispatch();
    const [products,selectedProduct] = useSelector(({productsReducer}) => [productsReducer.products,productsReducer.selectedProduct]);

    useEffect(() => {
        productActions.getProducts(dispatch);
    },[]);

    const handleFormChange = (event) => {
        setFormData({
            ...formData,
            [event.target.name]: event.target.value
            });
    }

    const handleDelete = async () => {
        productActions.deleteProduct(dispatch, selectedProduct.id);
    }

    const handleSelect = (product) => {
        if (selectedProduct.id === product.id)
            return;

        if (Object.keys(selectedProduct).length > 0) {
            productActions.cancelModifications(dispatch);
        }
        
        productActions.selectProduct(dispatch,product);
    }

    const handleModification = (event) => {
        const {name,value} = event.target;
        productActions.handleModifications(dispatch,{name,value});
    }

    const handleCancel = () => {
        productActions.cancelModifications(dispatch);
    }

    const handleSave = async () => {
        const index = products.findIndex((prod) => prod.id === selectedProduct.id);
        productActions.updateProduct(dispatch,products[index]);
    }

    const handleFormSubmit = (event) => {
        event.preventDefault();
        postForm.current.reset();
    }

    const handleAddProduct = async () => {
        const {name,price} = formData;

        if(name.length < 3 && price <= 0)
            return;

        productActions.addNewProduct(dispatch,formData);
    }


    return (<React.Fragment>
        <form
        ref={postForm}
        onChange={handleFormChange}
        onSubmit={handleFormSubmit}>
        <table>
            <thead>
                <tr>
                    <th>Name</th>
                    <th>Price</th>
                    <th>Action</th>
                </tr>
            </thead>
            <tbody>
                <tr>
                    <td>
                        <input name="name" type="text" placeholder="Product name"/>
                    </td>
                    <td>
                        <input name="price" type="number" placeholder="Price"/>
                    </td>
                    <td>
                        <button onClick={handleAddProduct}>Add Product</button>
                    </td>
                </tr>
            </tbody>
        </table>    
        </form>

        <table>
            <tbody>
                {products.map((product) => <TableEditableRow
                                        onSelect={()=>{handleSelect(product)}}
                                        onSave={handleSave}
                                        onCancel={handleCancel}
                                        onChange={handleModification}
                                        onDelete={handleDelete}
                                        selected={product.id === selectedProduct.id}
                                        values={product}
                                        key={product.id} />)}
            </tbody>
        </table>
    </React.Fragment>);
}

export default AdminTable;
```

In acest moment putem importa AdminTable.jsx in Dashboard.jsx

### Dashboard.jsx

```javascript
import React,{useEffect, useState,useRef} from 'react';
import {Redirect} from 'react-router-dom';
import Endpoints from './endpoints';
import AdminTable from './components/AdminTable/AdminTable';



const Dashboard = () => {
    
    const [redirect,setRedirect] = useState(null);
    const [loading, setLoading] = useState(true);
    

    useEffect(() => {
        const authenticate = async () => {
            try {
                const {data} = await Endpoints.login.authenticate();
                setRedirect(data.message !== "ok");
                setLoading(false);
            }
            catch(error) {
                setRedirect(true);
                setLoading(false);
            }
        }
        authenticate();
    },[]);

    const handleDisconnect = async () => {
        try {
            await Endpoints.login.disconnect();
            setRedirect(true);
        }
        catch (error) {
            console.log(error);
            setRedirect(true);
        }
    }

    if (loading) {
        return <React.Fragment>Loading...</React.Fragment>
    }

    if (redirect) {
        return <Redirect to='/login'/>
    }

    return (<React.Fragment>
        Dashboard
        <button onClick={handleDisconnect}>Disconnect</button>
        <AdminTable/>
    </React.Fragment>);
}

export default Dashboard;
```
