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
        axios.put('http://localhost:8080/product',{...selectedProduct},{withCredentials:true})
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
        try {
            await Endpoints.product.put(selectedProduct);
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
