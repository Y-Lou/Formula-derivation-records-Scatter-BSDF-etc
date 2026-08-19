# 问题解析  
---  
在现有的功能实现中，`CATFeatureImportAgent`代理会强制将所选的拓扑数据转换为`CATISpecObject_var`特征类型，因此会导致在获取路径信息时，会直接引用至原始的拓扑数据比如：  
```  
对一个曲面做提取操作，再将提取的面做接合，然后隐藏曲面数据，在选择接合上的面作为皮纹的话，高亮会跑到曲面上  
路径显示为：Part1-几何图形集.1-曲面.1-面.3  
正常希望的路径为：Part1-几何图形集.4-接合.1-面  
因此会因为路径问题导致，高亮出错从而形成误判  
```  
# 解决方案  
---  
- 将`CATFeatureImportAgent`代理替换为`CATPathElementAgent`代理  
- 对应设置  
    - ```c++  
        m_pfiaSurfaceSpecHL = new CATPathElementAgent("fiaSurfaceSpecHL");//创建类的指针  
        m_pfiaSurfaceSpecHL->AddElementType("CATSurface");//设置代理接收的选择类型  
        m_pfiaSurfaceSpecHL->AddElementType("CATIMfBiDimResult");//设置代理接收的选择类型  
        m_pfiaSurfaceSpecHL->SetBehavior(CATDlgEngWithPrevaluation | CATDlgEngWithCSO | CATDlgEngOneShot);//设置触发条件  
      ```  
- 高亮类型获取方案  
    - ```c++  
        //直接高亮  
        CATPathElement *pPathElment = m_pfiaSurfaceSpecHL->GetValue();  
        pPathElment->InitToLeafElement();  
        m_pHSO->AddElement(pPathElement);  
        //重新打开特征高亮显示  
        /*  
        需要代理中存放的CATSurface的唯一Tag  
        需要获取到代理中存放的父级特征  
        */  
        CATPathElement *pPathElment = m_pfiaSurfaceSpecHL->GetValue();  
        pPathElment->InitToLeafElement();  
        CATSurface *pMfBrep1 = NULL;//CATSurface指针  
        hr = pPathElment->Search(IID_CATSurface,(void**)&pMfBrep1 );//获取选择的面  
        if (SUCCEEDED(hr) && pMfBrep1)//判空  
        {  
            cout << "Succeed" << pMfBrep1->GetPersistentTag() << endl;  
            CATULONG32 tagId = pMfBrep1->GetPersistentTag();//获取tag  
        }  
        CATISpecObject* findspeobj = NULL;//CATISpecObject指针  
        hr = pPathElment->Search(IID_CATISpecObject,(void**)&findspeobj );//获取CATISpecObject  
        //创建CATISpecObject的CATPathElement  
        CATIBuildPath* piBuildPath = nullptr;  
        CATFrmEditor* pEditor = CATFrmEditor::GetCurrentEditor();  
        // 1. 获取 CATIBuildPath 接口  
        HRESULT rc = findspeobj->QueryInterface(IID_CATIBuildPath, (void**)&piBuildPath);  
        if (FAILED(rc) || !piBuildPath) {  
            return;  
        }  
        // 2. 获取上下文路径 (通常是当前UI激活的对象的路径)[reference:4]  
        CATPathElement contextPath = pEditor->GetUIActiveObject();  
        // 3. 提取路径[reference:5]  
        rc = piBuildPath->ExtractPathElement(&contextPath, &pPathElement);  
        piBuildPath->Release();  
        if (SUCCEEDED(rc) && pPathElement) {  
            cout << pPathElement->GetSize() << endl;  
            pPathElement->AddChildElement(pMfBrepf2);  
            cout << pPathElement->GetSize() << endl;  
            m_pHSO->AddElement(pPathElement);  
        }  
      ```  
- 名称更新，获取`CATSurface`的名称与`CATISpecObject`的名称做拼接  
# 新增成员变量  
---  
## 特征中  
---  
添加`SKM::ListV<long int> tagList`//用于存储面片的tag，用于模拟判断是否为皮纹面  
添加`CATListValCATISpecObject_var FatherRootList`//用于存储`CATSurface`的父节点，用于高亮显示总的数据接口(中间数据处理)  
## CMD类中  
---  
- `CATLISTP(CATPathElement) m_PathElementList` 用于高亮  
## 修改逻辑  
---  
1. 在原有`SurfaceList`有值时  
    2. 在`CMD`类中对原有的`SurfaceList`做提取`tag`的操作，存入`tagList`中  
    3. 在`CMD`类中对原有的`SurfaceList`做提取`FatherRoot`的操作存入`FatherRootList`中  
    4. 在`CMD`类中对原有的`SurfaceList`做提取`CATPathElement`存入`m_PathElementList`中  
5. 为空时  
    6. 直接使用`tagList`和`FatherRootList`来生成`m_PathElementList`  
# 总结  
---  
`CATSurface`类型的`PersistentTag`并不唯一，`CATFace`是唯一的，因此上述方法有问题。  
  
---  
# 方案二  
---  
测试发现，`CATFeatureImportAgent`在鼠标预选时会获得正确的`CATPathElement`路径，因此获取预路径作为高亮，点选路径进行保存即可。  
关于二次打开重新高亮的问题，可以保存预选`CATPathElement`中的`CATBaseUnknown_var`保存到`CATListValCATBaseUnknown_var`中存储到特征里，并将`CATPathElement`中的父级特征提取出来存储到`CATListValCATISpecObject_var`中，进行拼接后高亮显示。  
# 方案三  
---  
使用两个`CATFeatureImportAgent`代理，一个接收鼠标预选信号，一个接收鼠标点击信号，通过数据判断的形式进行`CATFace->Tag`、`FatherRoot`、`SpecObject`的存储与删除，新建`PathElement`进行高亮，**可实现鼠标滑动多选功能**  
## CATFeatureImportAgent设置  
---  
```c++  
// 基础设置  
{  
    m_pfiaSurfaceSpec = new CATFeatureImportAgent("fiaLightGuideBodySpec");  
    m_pfiaSurfaceSpec->SetOrderedElementType("CATSurface");//设置选择类型  
    m_pfiaSurfaceSpec->SetBehavior(CATDlgEngWithPrevaluation |CATDlgEngAcceptOnPrevaluate| CATDlgEngWithCSO );//设置触发响应条件(鼠标预选触发)  
    m_pfiaSurfaceSpec->SetAgentBehavior(MfPermanentBody | MfLastFeatureSupport | MfRelimitedFeaturization );//设置代理内部动作  
      
    m_pfiaSurfaceSpecActivate = new CATFeatureImportAgent("fiaLightGuideBodySpec",NULL,NULL,MfNoDuplicateFeature);  
    m_pfiaSurfaceSpecActivate->SetOrderedElementType("CATSurface");//设置选择类型  
    m_pfiaSurfaceSpecActivate->SetBehavior(CATDlgEngWithCSO | CATDlgEngOneShot | CATDlgEngMultiAcquisitionSelModes);//设置触发响应条件(鼠标点选触发)  
    m_pfiaSurfaceSpecActivate->SetAgentBehavior(MfPermanentBody | MfLastFeatureSupport | MfRelimitedFeaturization );//设置代理内部动作  
    /*  
    注意点：  
    两个代理的响应条件必须不一致，如果一致会导致响应触发冲突  
    */  
}  
// 状态机设置  
{  
    //状态机类型   CATDialogState  
    //点选状态机  
    m_WaitForObjectState = GetInitialState("Select a Curve or another input field");  
    m_WaitForObjectState->AddDialogAgent(m_pfiaSurfaceSpecActivate);  
    //预选状态机  
    m_WaitForObjectStateActivate = GetInitialState("Select a Curve");  
    m_WaitForObjectStateActivate->AddDialogAgent(m_pfiaSurfaceSpec);  
    /*  
    注意点：  
    两个代理需要绑定至不同的状态机中，防止响应冲突  
    */  
}  
// 响应函数绑定  
{  
    //预选响应函数  
    AddTransition(m_WaitForObjectStateActivate, m_WaitForObjectStateActivate, IsLastModifiedAgentCondition(m_pfiaSurfaceSpec),  
        Action((ActionMethod)&HVCSOPPropertiesMainCmd::fiaSurfaceSpecSelected));  
    //点选形影函数  
    AddTransition(m_WaitForObjectState, m_WaitForObjectState, IsOutputSetCondition(m_pfiaSurfaceSpecActivate),  
        Action((ActionMethod)&HVCSOPPropertiesMainCmd::fiaSurfaceSpecSelectedActivate));  
}  
```  
## 预选响应函数  
---  
```c++  
/*  
该响应函数分为两个功能  
1、只获取预选的CATPathElement，用于后续点选创建高亮  
2、鼠标滑动多选，对预选的CATPathElement进行手动CATIFeaturize来保存需要的特征  
    1、鼠标滑动多选需要两个状态进行控制  
        1、m_pMainDlg->_CheckButtonMultipleChoice->GetState() == CATDlgCheck 复选框状态代表功能启用  
        2、m_MultipleChoiceIsOn布尔变量代表选择开始  
3、因为响应很灵敏，为了方便操作添加1s的延迟  
*/  
void HVCSOPPropertiesMainCmd::fiaSurfaceSpecSelected()  
{  
    CATFrmEditor  * pEditor = CATFrmEditor::GetCurrentEditor();  
    if(NULL != pEditor)      
    {  
        CATPSO* pPSO = pEditor->GetPSO();  
        int size = pPSO->GetSize();  
        if (size > 0)  
        {  
            CATPathElement * pPath = (CATPathElement*) (*pPSO)[size-1] ;  
            pPSO->Empty();  
            pPSO->AddElement(pPath);  
        }  
        else  
        {  
            pPSO->Empty();  
        }  
          
    }  
    HRESULT hr = E_FAIL;  
    m_PathElementPreactivate = *m_pfiaSurfaceSpec->GetValue();  
    m_PathElementPreactivate.InitToLeafElement();  
    m_pfiaSurfaceSpec->InitializeAcquisition();  
    //如果启动多选则进行手动CATIFeaturize  
    if (m_pMainDlg->_CheckButtonMultipleChoice->GetState() == CATDlgCheck && m_MultipleChoiceIsOn == true)  
    {  
        // 代理太过灵敏使用时间间隔来判断是否进行选择间隔1s  
        time_t start = m_time;  
        time(&m_time);  
        cout << "delte time = " << difftime(m_time,start) << endl;  
        if (difftime(m_time,start) < 1) return;  
        //step1：先进行CATIFeaturize获取特征  
        CATISpecObject_var spResult = NULL_var;  
        if (NULL == &m_PathElementPreactivate)return;  
        if (m_PathElementPreactivate.GetSize() <= 0)return;  
        // 取得 Path 的叶子对象  
        const int leafIndex = m_PathElementPreactivate.GetSize() - 1;  
        CATBaseUnknown* pLeaf = (m_PathElementPreactivate)[leafIndex];  
        if (NULL == pLeaf)return;  
        // 尝试取得 CATIFeaturize  
        CATIFeaturize_var spFeaturize = pLeaf;  
        if (NULL_var == spFeaturize)return;  
        // 执行 Featurization  
        spResult = spFeaturize->FeaturizeR(  
            MfPermanentBody  
            | MfLastFeatureSupport  
            | MfRelimitedFeaturization  
            );  
        if (NULL_var != spResult)  
        {  
            // Featurize 后必须 Update  
            CATTry  
            {  
                spResult->Update();  
            }  
            CATCatch(CATError, error)  
            {  
                ::Flush(error);  
                spResult = NULL_var;  
            }  
            CATEndTry;  
        }  
        //step2：获取父节点和CATFace-tag，进行判断，并存储或者去除  
        //获取父节点  
        CATISpecObject* findspeobj = NULL;  
        hr = m_PathElementPreactivate.Search(IID_CATISpecObject,(void**)&findspeobj );  
        if (FAILED(hr))    return;  
        if (!findspeobj) return;  
        if (m_ParamBase.FatherRootList.Locate(findspeobj)==0)  
        {  
            m_ParamBase.FatherRootList.Append(findspeobj);  
            cout << "Find New Father Root" << endl;  
        }  
        //获取CATFace-tag  
        CATISpecObject_var specObj = spResult;  
        CATBody_var spBody;  
        spBody = SKC::GetBodyFromFeature(specObj);//get body  
        if (spBody == NULL_var) return; //无法获得指针，跳过这个body  
        CATLISTP(CATCell) cells;//用于离散的cell列表，此程序用于曲面离散，故cells的大小为一。  
        spBody->GetAllCells(cells,2);//获取cell  
        int numberOfCells = cells.Size();//获取size  
        if (numberOfCells <= 0) return;  
        CATFace * prFace = NULL;  
        for (int ii = 1; ii <= cells.Size(); ii++)  
        {  
            // 获取tag  
            CATCell* cell = cells[ii];  
            prFace = (CATFace*) cell;  
            CATULONG32 tagNum = prFace->GetPersistentTag();  
            CATUnicodeString tagStr = "";  
            tagStr.BuildFromNum(tagNum);  
            int IndexTag = m_ParamBase.tagList.Locate(tagStr);  
            if (IndexTag > 0)  
            {  
                m_ParamBase.tagList.RemovePosition(IndexTag);  
                m_ParamBase.SurfaceList.RemovePosition(IndexTag);  
                m_PathElementList.RemovePosition(IndexTag);  
            }  
            else  
            {  
                m_ParamBase.tagList.Append(tagStr);  
                m_ParamBase.SurfaceList.Append(specObj);  
                //创建PathElement  
                CATFrmEditor* pEditor = CATFrmEditor::GetCurrentEditor();  
                if (m_pPathElementPreactivate) m_pPathElementPreactivate->Release();  
                m_pPathElementPreactivate = NULL;  
                CATIBuildPath* piBuildPath = NULL;  
                HRESULT rc = findspeobj->QueryInterface(IID_CATIBuildPath, (void**)&piBuildPath);  
                if (FAILED(rc) || !piBuildPath) {  
                    return;  
                }  
                CATPathElement contextPath = pEditor->GetUIActiveObject();  
                hr = piBuildPath->ExtractPathElement(&contextPath, &m_pPathElementPreactivate);  
                piBuildPath->Release();  
                CATPathElement * newElement = NULL;  
                CATIMechanicalFeature *piMechanicalFeature = NULL;  
                hr = findspeobj->QueryInterface( IID_CATIMechanicalFeature,(void**) &piMechanicalFeature );  
                if ( FAILED(hr) )return;  
                newElement = piMechanicalFeature->BuildPathElement(m_pPathElementPreactivate, cell);  
                m_PathElementList.Append(newElement);  
            }  
        }  
        //step3：高亮  
        m_pHSO->Empty();  
        // 高亮  
        for (int i = 1; i <= m_PathElementList.Size(); i++)  
        {  
            m_pHSO->AddElement(m_PathElementList[i]);  
        }  
    }  
}  
```  
## 点选响应函数  
---  
```c++  
/*  
1、通过预选的CATPathElement和点选的CATSpecObject进行正确高亮元素的创建  
2、控制鼠标滑动多选的开始/结束状态切换控制  
3、保存对应的FatherRoot、Tag、CATSpecObject  
*/  
void HVCSOPPropertiesMainCmd::fiaSurfaceSpecSelectedActivate()  
{  
    m_pHSO->Empty();  
    CATListValCATISpecObject_var oPointList;  
    HRESULT hr = SKC::GetSelectedObjects(m_pfiaSurfaceSpecActivate, oPointList);  
    m_pfiaSurfaceSpecActivate->InitializeAcquisition();  
    if (m_pMainDlg->_CheckButtonMultipleChoice->GetState() == CATDlgCheck)  
    {  
        if (m_MultipleChoiceIsOn)  
        {  
            m_MultipleChoiceIsOn = false;  
        }  
        else  
        {  
            m_MultipleChoiceIsOn = true;  
        }  
        ////准确性测试  
        //{  
        //    //大小判断  
        //    cout << "Surface size = " << m_ParamBase.SurfaceList.Size() << " tag size = " << m_ParamBase.tagList.Size() << endl;  
        //    //tag数据判断  
        //    for (int i = 1; i <= m_ParamBase.SurfaceList.Size(); i++)  
        //    {  
        //        CATISpecObject_var specObj = m_ParamBase.SurfaceList[i];  
        //        CATBody_var spBody;  
        //        spBody = SKC::GetBodyFromFeature(specObj);//get body  
        //        if (spBody == NULL_var) return; //无法获得指针，跳过这个body  
        //        CATLISTP(CATCell) cells;//用于离散的cell列表，此程序用于曲面离散，故cells的大小为一。  
        //        spBody->GetAllCells(cells,2);//获取cell  
        //        int numberOfCells = cells.Size();//获取size  
        //        if (numberOfCells <= 0) return;  
        //        CATFace * prFace = NULL;  
        //        CATULONG32 tagNum;  
        //        for (int ii = 1; ii <= cells.Size(); ii++)  
        //        {  
        //            // 获取tag  
        //            CATCell* cell = cells[ii];  
        //            prFace = (CATFace*) cell;  
        //            tagNum = prFace->GetPersistentTag();  
        //        }  
        //        cout << "Surface tag = " << tagNum << " tag = " << m_ParamBase.tagList[i] << endl;  
        //    }  
        //}  
        if(!m_MultipleChoiceIsOn)  
        {  
            m_pHSO->Empty();  
            // 高亮  
            for (int i = 1; i <= m_PathElementList.Size(); i++)  
            {  
                m_pHSO->AddElement(m_PathElementList[i]);  
            }  
            m_spSopPropertiesBase->SetParamInfors(m_ParamBase);  
            m_pMainDlg->SetPathElementList(m_PathElementList);  
            m_pMainDlg->UpdateDialog();  
            m_pMainDlg->SetActiveSListFocus();  
            UpdateFaceCount();//更新数量显示  
        }  
        return;  
    }  
    SKC:: CATListValCATISpecObjectSetValue(m_ParamBase.SurfaceList, oPointList, ElementNormal);//去重和反选  
    // 先实现单选  
    // 记录父节点以及高亮使用的PathElement  
    CATISpecObject* findspeobj = NULL;  
    hr = m_PathElementPreactivate.Search(IID_CATISpecObject,(void**)&findspeobj );  
    if (FAILED(hr))    return;      
    //判断父级是否重复  
    if (!findspeobj) return;  
    if (m_ParamBase.FatherRootList.Locate(findspeobj)==0)  
    {  
        m_ParamBase.FatherRootList.Append(findspeobj);  
        cout << "Find New Father Root" << endl;  
    }  
    //记录CATFace的TAG  
    for (int i = 1; i <= oPointList.Size(); i++)  
    {  
        CATISpecObject_var specObj = oPointList[i];  
        CATBody_var spBody;  
        spBody = SKC::GetBodyFromFeature(specObj);//get body  
        if (spBody == NULL_var) return; //无法获得指针，跳过这个body  
        CATLISTP(CATCell) cells;//用于离散的cell列表，此程序用于曲面离散，故cells的大小为一。  
        spBody->GetAllCells(cells,2);//获取cell  
        int numberOfCells = cells.Size();//获取size  
        if (numberOfCells <= 0) return;  
        CATFace * prFace = NULL;  
        for (int ii = 1; ii <= cells.Size(); ii++)  
        {  
            // 获取tag  
            CATCell* cell = cells[ii];  
            prFace = (CATFace*) cell;  
            CATULONG32 tagNum = prFace->GetPersistentTag();  
            CATUnicodeString tagStr = "";  
            tagStr.BuildFromNum(tagNum);  
            //cout << "surface is = " << prFace->GetSurface()->GetPersistentTag() << endl;  
            int IndexTag = m_ParamBase.tagList.Locate(tagStr);  
            if (IndexTag > 0)  
            {  
                m_ParamBase.tagList.RemovePosition(IndexTag);  
                m_PathElementList.RemovePosition(IndexTag);  
            }  
            else  
            {  
                m_ParamBase.tagList.Append(tagStr);  
                //创建PathElement  
                CATFrmEditor* pEditor = CATFrmEditor::GetCurrentEditor();  
                if (m_pPathElementPreactivate) m_pPathElementPreactivate->Release();  
                m_pPathElementPreactivate = NULL;  
                CATIBuildPath* piBuildPath = NULL;  
                HRESULT rc = findspeobj->QueryInterface(IID_CATIBuildPath, (void**)&piBuildPath);  
                if (FAILED(rc) || !piBuildPath) {  
                    return;  
                }  
                CATPathElement contextPath = pEditor->GetUIActiveObject();  
                hr = piBuildPath->ExtractPathElement(&contextPath, &m_pPathElementPreactivate);  
                piBuildPath->Release();  
                CATPathElement * newElement = NULL;  
                CATIMechanicalFeature *piMechanicalFeature = NULL;  
                hr = findspeobj->QueryInterface( IID_CATIMechanicalFeature,(void**) &piMechanicalFeature );  
                if ( FAILED(hr) )return;  
                newElement = piMechanicalFeature->BuildPathElement(m_pPathElementPreactivate, cell);  
                m_PathElementList.Append(newElement);  
            }  
        }  
    }  
  
    //准确性测试  
    //{  
    //    //大小判断  
    //    cout << "Surface size = " << m_ParamBase.SurfaceList.Size() << " tag size = " << m_ParamBase.tagList.Size() << endl;  
    //    //tag数据判断  
    //    for (int i = 1; i <= m_ParamBase.SurfaceList.Size(); i++)  
    //    {  
    //        CATISpecObject_var specObj = oPointList[i];  
    //        CATBody_var spBody;  
    //        spBody = SKC::GetBodyFromFeature(specObj);//get body  
    //        if (spBody == NULL_var) return; //无法获得指针，跳过这个body  
    //        CATLISTP(CATCell) cells;//用于离散的cell列表，此程序用于曲面离散，故cells的大小为一。  
    //        spBody->GetAllCells(cells,2);//获取cell  
    //        int numberOfCells = cells.Size();//获取size  
    //        if (numberOfCells <= 0) return;  
    //        CATFace * prFace = NULL;  
    //        CATULONG32 tagNum;  
    //        for (int ii = 1; ii <= cells.Size(); ii++)  
    //        {  
    //            // 获取tag  
    //            CATCell* cell = cells[ii];  
    //            prFace = (CATFace*) cell;  
    //            tagNum = prFace->GetPersistentTag();  
    //        }  
    //        cout << "Surface tag = " << tagNum << " tag = " << m_ParamBase.tagList[i] << endl;  
    //    }  
    //}  
  
    // 高亮  
    for (int i = 1; i <= m_PathElementList.Size(); i++)  
    {  
        m_pHSO->AddElement(m_PathElementList[i]);  
    }  
    m_spSopPropertiesBase->SetParamInfors(m_ParamBase);  
    m_pMainDlg->SetPathElementList(m_PathElementList);  
    m_pMainDlg->UpdateDialog();  
    m_pMainDlg->SetActiveSListFocus();  
    UpdateFaceCount();//更新数量显示  
}  
```