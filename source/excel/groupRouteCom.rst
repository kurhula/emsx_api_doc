#########################
Group Route Request (COM)
#########################


.. code-block:: vb.net

    Option Explicit

    Private WithEvents m_BBG_EMSX As blpapicomLib2.Session
    Public running As Boolean
    Private svc As blpapicomLib2.service
    Private emsxService As String
    Private requestID As blpapicomLib2.CorrelationId

    Private Sub Class_Initialize()

        log "Bloomberg - EMSX API Example - GroupRoute"

        emsxService = "//blp/emapisvc_beta"
        
        Set m_BBG_EMSX = New blpapicomLib2.Session
        
        running = True
        
        m_BBG_EMSX.QueueEvents = True
        m_BBG_EMSX.Start
        

    End Sub

    Private Sub Class_Terminate()
        Set m_BBG_EMSX = Nothing
    End Sub

    Private Sub m_BBG_EMSX_ProcessEvent(ByVal obj As Object)

        On Error GoTo errHandler

        Dim eventObj As blpapicomLib2.Event
        
         '   Assign the returned data to a Bloomberg type event
        Set eventObj = obj
        
        If Application.Ready Then
        
            Select Case eventObj.EventType
            
                Case SESSION_STATUS
                    processSessionEvent eventObj
                    
                Case BLPSERVICE_STATUS
                    processServiceEvent eventObj
                    
                Case RESPONSE
                    processResponseEvent eventObj
                    
            End Select
            
        End If

        Exit Sub

    errHandler:
        Dim errmsg As Variant
        errmsg = Err.Description
        log (errmsg)
        running = False

    End Sub


    Private Sub processSessionEvent(evt As blpapicomLib2.Event)

        log "Processing SESSION_STATUS event"
        
        Dim it As blpapicomLib2.MessageIterator
        
        Set it = evt.CreateMessageIterator()

        ' Loop while we have messages remaining
        Do While it.Next()
                  
            Dim msg As Message
            
            '   Pick up message
            Set msg = it.Message
            
            log "MessageType: " + msg.MessageTypeAsString
            
            If msg.MessageTypeAsString = "SessionStarted" Then
                log "Session started..."
                m_BBG_EMSX.OpenService emsxService
            ElseIf msg.MessageTypeAsString = "SessionStartupFailure" Then
                log "Error: Session startup failed"
                running = False
            End If
            
        Loop

    End Sub

    Private Sub processServiceEvent(evt As blpapicomLib2.Event)

        Dim req As REQUEST
        Dim service As service
        Dim strategy As Element
        Dim indicator As Element
        Dim data As Element
        Dim it As blpapicomLib2.MessageIterator
        
        On Error GoTo failed
        
        log "Processing SERVICE_STATUS event"
        
        Set it = evt.CreateMessageIterator()

        ' Loop while we have messages remaining
        Do While it.Next()
                  
            Dim msg As Message
            
            '   Pick up message
            Set msg = it.Message
            
            log "MessageType: " + msg.MessageTypeAsString
            
            If msg.MessageTypeAsString = "ServiceOpened" Then
        
                ' Get the service
                Set service = m_BBG_EMSX.GetService(emsxService)
        
                'First, create our request object
                Set req = service.CreateRequest("GroupRouteEx")
                
                'Multiple order numbers can be added
                req.Append "EMSX_SEQUENCE", 3741540
                req.Append "EMSX_SEQUENCE", 3741541
                req.Append "EMSX_SEQUENCE", 3741542
        
                'The fields below are mandatory
                req.Set "EMSX_AMOUNT_PERCENT", 100
                req.Set "EMSX_BROKER", "BMTB"
                
                'For GroupRoute, the below values need to be added, but are taken
                'from the original order when the route is created.
                req.Set "EMSX_HAND_INSTRUCTION", "ANY"
                req.Set "EMSX_ORDER_TYPE", "MKT"
                req.Set "EMSX_TICKER", "IBM US Equity"  'Use to identify the asset class of all orders
                req.Set "EMSX_TIF", "DAY"

                'The fields below are optional
                'req.Set "EMSX_ACCOUNT", "TestAccount"
                'req.Set "EMSX_BOOKNAME", "HedgingBasket"
                'req.Set "EMSX_CFD_FLAG", "1"
                'req.Set "EMSX_CLEARING_ACCOUNT", "ClrAccName"
                'req.Set "EMSX_CLEARING_FIRM", "FirmName"
                'req.Set "EMSX_EXEC_INSTRUCTIONS", "AnyInst"
                'req.Set "EMSX_GET_WARNINGS", "0"
                'req.Set "EMSX_GTD_DATE", "20170105"
                'req.Set "EMSX_LIMIT_PRICE", 123.45
                'req.Set "EMSX_LOCATE_BROKER", "BMTB"
                'req.Set "EMSX_LOCATE_ID", "SomeID"
                'req.Set "EMSX_LOCATE_REQ", "Y"
                'req.Set "EMSX_NOTES", "Some notes"
                'req.Set "EMSX_ODD_LOT", "0"
                'req.Set "EMSX_P_A", "P"
                'req.Set "EMSX_RELEASE_TIME", 34341
                'req.Set "EMSX_REQUEST_SEQ", 1001
                'req.Set "EMSX_STOP_PRICE", 123.5
                'req.Set "EMSX_TRADER_UUID", 1234567
               
                'Set the Request Type if this is for multi-leg orders
                'only valid for options
                
                'Dim requestType As Element
                '
                'requestType = req.GetElement("EMSX_REQUEST_TYPE")
                'requestType.SetChoice "Multileg"
                '
                'Dim multileg As Element
                'multileg = requestType.GetElement("Multileg")
                'multileg.SetElement "EMSX_AMOUNT", 10
                'multileg.GetElement("EMSX_ML_RATIO").AppendValue 2
                'multileg.GetElement("EMSX_ML_RATIO").AppendValue 3

                'Add the Route Ref ID values
                Dim routeRefIDPairs As Element
                Set routeRefIDPairs = req.GetElement("EMSX_ROUTE_REF_ID_PAIRS")
                        
                Dim route1 As Element
                Set route1 = routeRefIDPairs.AppendElment()
                route1.SetElement "EMSX_ROUTE_REF_ID", "MyRouteRef1"
                route1.SetElement "EMSX_SEQUENCE", 3741540
                        
                Dim route2 As Element
                Set route2 = routeRefIDPairs.AppendElment()
                route2.SetElement "EMSX_ROUTE_REF_ID", "MyRouteRef2"
                route2.SetElement "EMSX_SEQUENCE", 3741541
                        
                Dim route3 As Element
                Set route3 = routeRefIDPairs.AppendElment()
                route3.SetElement "EMSX_ROUTE_REF_ID", "MyRouteRef3"
                route3.SetElement "EMSX_SEQUENCE", 3741542
                        
                'Below we establish the strategy details. Strategy details
                'are common across all orders in a GroupRoute operation.
                Set strategy = req.GetElement("EMSX_STRATEGY_PARAMS")
                strategy.SetElement "EMSX_STRATEGY_NAME", "VWAP"
                
                Set indicator = strategy.GetElement("EMSX_STRATEGY_FIELD_INDICATORS")
                Set data = strategy.GetElement("EMSX_STRATEGY_FIELDS")
                            
                'Strategy parameters must be appended in the correct order. See the output
                'of GetBrokerStrategyInfo request for the order. The indicator value is 0 for
                'a field that carries a value, and 1 where the field should be ignored
                
                data.AppendElment().SetElement "EMSX_FIELD_DATA", "09:30:00"    'StartTime
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 0

                data.AppendElment().SetElement "EMSX_FIELD_DATA", "10:30:00"   'EndTime
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 0

                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           'Max%Volume
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1

                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           '%AMSession
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1

                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           'OPG
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1

                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           'MOC
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1

                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           'CompletePX
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1
                        
                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           'TriggerPX
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1

                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           'DarkComplete
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1

                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           'DarkCompPX
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1

                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           'RefIndex
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1

                data.AppendElment().SetElement "EMSX_FIELD_DATA", ""           'Discretion
                indicator.AppendElment().SetElement "EMSX_FIELD_INDICATOR", 1
                
                log "Request: " & req.Print
                
                ' Send the request
                Set requestID = m_BBG_EMSX.SendRequest(req)

            ElseIf msg.MessageTypeAsString = "ServiceOpenFailure" Then
            
                log "Error: Service failed to open"
                running = False
                
            End If
            
        Loop

        Exit Sub
        
    failed:

        log "Failed to send the request: " + Err.Description
        
        running = False
        Exit Sub
        
    End Sub

    Private Sub processResponseEvent(evt As blpapicomLib2.Event)

        log "Processing RESPONSE event"
        
        Dim it As blpapicomLib2.MessageIterator
        Dim i As Integer
        Dim errorCode As Long
        Dim errorMessage As String
     
        Set it = evt.CreateMessageIterator()

        ' Loop while we have messages remaining
        Do While it.Next()
                  
            Dim msg As Message
            
            '   Pick up message
            Set msg = it.Message
            
            log "MessageType: " + msg.MessageTypeAsString
            
            If evt.EventType = RESPONSE And msg.CorrelationId.Value = requestID.Value Then
            
                If msg.MessageTypeAsString = "ErrorInfo" Then
                
                    errorCode = msg.GetElement("ERROR_CODE")
                    errorMessage = msg.GetElement("ERROR_MESSAGE")
                    
                    log "ERROR CODE: " & errorCode & "    ERROR DESCRIPTION: " & errorMessage
                
                    running = False
                    
                ElseIf msg.MessageTypeAsString = "GroupRouteEx" Then
                    
                    Dim numValues As Integer
                    Dim emsxSequence As Long
                    Dim emsxRouteId As Long
                    Dim e As Element
                    
                    If msg.AsElement.HasElement("EMSX_SUCCESS_ROUTES") Then
                    
                        Dim success As Element
                        
                        Set success = msg.GetElement("EMSX_SUCCESS_ROUTES")

                        numValues = success.numValues
                                
                        For i = 0 To numValues - 1
                                    
                            Set e = success.GetValueAsElement(i)
                            
                            emsxSequence = e.GetElement("EMSX_SEQUENCE")
                            emsxRouteId = e.GetElement("EMSX_ROUTE_ID")
                            
                            log "Success: " & emsxSequence & ", " & emsxRouteId
                        Next i
                    End If
                    
                    If msg.AsElement.HasElement("EMSX_FAILED_ROUTES") Then
                        
                        Dim failed As Element
                        
                        Set failed = msg.GetElement("EMSX_FAILED_ROUTES")

                        numValues = failed.numValues
                                
                        For i = 0 To numValues - 1
                                    
                            Set e = failed.GetValueAsElement(i)
                            
                            emsxSequence = e.GetElement("EMSX_SEQUENCE")
                            errorCode = e.GetElement("ERROR_CODE")
                            errorMessage = e.GetElement("ERROR_MESSAGE")
                            
                            log "Failed: " & emsxSequence & ", " & errorCode & ", " & errorMessage
                            
                        Next i
                    End If
                    
                    m_BBG_EMSX.Stop
                    running = False
                
                End If
            End If
        Loop

    End Sub




