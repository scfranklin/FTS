# FTS
function [PortValue, ReturnIndex] = VortexCalculatePortfolio( backtest_start, backtest_end, Rebal_date, OptionClosestToLowDeltaSP,OptionClosestToHighDeltaSP, OptionClosestToXDeltaVIX,SPHighDelta,SPLowDelta, VIXDelta)

%convert to serial numbers
backtest_start = {'12/16/2010'};
backtest_end = {'2/17/2015'};
backtest_start_serial = datenum(backtest_start);
backtest_end_serial = datenum(backtest_end);
rebal_date_serial = datenum(Rebal_date);

fields_ask = 'PX_ASK';
fields_bid = 'PX_BID';
fields_last = 'PX_LAST';
per = {'daily'};
cur = {'USD'};
%use ContractSizeSPLow = 0 if put only (not put spread)
ContractSizeSPLow = 100;
ContractSizeSPHigh = 100;
ContractSizeVIX = 100;
H = holidays(backtest_start_serial, backtest_end_serial); %identify holidays

%pull option values for applicable trading days

formatOut = 'mm/dd/yyyy';
PortValue = {};

%start with j = 1, j = 2, j = 3

PortNum = {'One' 'Two' 'Three'};
tic
for p = 1:length(PortNum)
    j = p;
    PortValue.(PortNum{p}) = [];
    backtest_start_serial = rebal_date_serial(j);
    k =1;
    for z = 1:length(H)
        if backtest_start_serial > H(k)
            k = k +  1;
        else
            continue
        end
    end
    OptionPriceSPLow = [];
    OptionPriceSPHigh = [];
    OptionPriceVIX = [];
    OptionNameSPLow = [];
    OptionNameSPHigh = [];
    OptionNameVIX = [];
    Ratios = zeros(1,4);
    SPContractHighPrice = [];
    SPContractLowPrice = [];
    VIXContractPrice = [];
    DataStore = [];
    ShortPutSpreadPrice = [];
    VIXCallPrice = [];
    PortReturn = [];
    NumVIXOptions = [];
    PercentChange=[];
    %     tempPortValue = [];
    for date_index = backtest_start_serial:backtest_end_serial %(backtest_end_serial-backtest_start_serial)
        [DayNumber, DayName] = weekday((date_index)); %identify day of week
        if DayNumber == 7 || DayNumber == 1 %exclude weekends
            continue
        elseif date_index == H(k) %exclude holidays
            if k == length(H)
                continue
            else
                k = k +1;
            end
            continue
        else
            if j+3 < length(rebal_date_serial) && date_index == rebal_date_serial(j+3) %if date of next rebalance day, move index forward
                j = j +3;
            end
            [tempSPPrice, tempSPName] = history(c, 'SPX Index', fields_last, {datestr(date_index,formatOut)}, {datestr(date_index,formatOut)}, per, cur);
            [tempVIXPrice, tempVIX] = history(c, 'VIX Index', fields_last, {datestr(date_index,formatOut)}, {datestr(date_index,formatOut)}, per, cur);
            
            % if rebalancing 
            if date_index == rebal_date_serial(j)            %for rebal_date, PX_BID (sell put spread), buy VIX call
                [tempOptionPriceSPHigh, tempSPHighName] = history(c, OptionClosestToHighDeltaSP(j,:), fields_bid, {datestr(date_index,formatOut)}, {datestr(date_index,formatOut)}, per, cur);
                [tempOptionPriceSPLow, tempSPLowName] = history(c, OptionClosestToLowDeltaSP(j,:), fields_ask, {datestr(date_index,formatOut)}, {datestr(date_index,formatOut)}, per, cur);
                [tempOptionPriceVIX, tempVIXName] = history(c, OptionClosestToXDeltaVIX(j,:), fields_ask, {datestr(date_index,formatOut)}, {datestr(date_index,formatOut)}, per, cur);
                
            % if not rebalancing 
            else %j+3 < length(rebal_date_serial) && date_index < rebal_date_serial(j+3) || j+3 >= length(rebal_date_serial) && date_index < rebal_date_serial(end)
                
                % get option prices if not rebalance day
                % get PX_ask value of put spread and PX_bid value of VIX
                % call
                [tempOptionPriceSPHigh, tempSPHighName] = history(c, OptionClosestToHighDeltaSP(j,:), fields_ask, {datestr(date_index,formatOut)}, {datestr(date_index,formatOut)}, per, cur);
                [tempOptionPriceSPLow, tempSPLowName] = history(c, OptionClosestToLowDeltaSP(j,:), fields_bid, {datestr(date_index,formatOut)}, {datestr(date_index,formatOut)}, per, cur);
                [tempOptionPriceVIX, tempVIXName] = history(c, OptionClosestToXDeltaVIX(j,:), fields_bid, {datestr(date_index,formatOut)}, {datestr(date_index,formatOut)}, per, cur);
                if size(tempOptionPriceSPHigh)==[0,0]
                    tempOptionPriceSPHigh = [date_index 0];
                end
                if size(tempOptionPriceSPLow)==[0,0]
                    tempOptionPriceSPLow = [date_index 0];
                end
                if size(tempOptionPriceVIX)==[0,0]
                    tempOptionPriceVIX = [date_index 0];
                end
            end
            
            OptionPriceSPLow = [OptionPriceSPLow; tempOptionPriceSPLow];
            OptionPriceSPHigh = [OptionPriceSPHigh; tempOptionPriceSPHigh];
            OptionPriceVIX = [OptionPriceVIX; tempOptionPriceVIX];
            OptionNameSPLow = [OptionNameSPLow; OptionClosestToLowDeltaSP(j,:)];
            OptionNameSPHigh = [OptionNameSPHigh; OptionClosestToHighDeltaSP(j,:)];
            OptionNameVIX = [OptionNameVIX; OptionClosestToXDeltaVIX(j,:)];
            
            % Option price as percent of index
            tempRatios = [date_index tempOptionPriceSPHigh(:,2)/tempSPPrice(:,2) tempOptionPriceSPLow(:,2)/tempSPPrice(:,2) tempOptionPriceVIX(:,2)/tempVIXPrice(:,2)];
            % if rebalancing, no change in returns
            if date_index == rebal_date_serial(j)
                tempPercentChange = zeros(1,size(Ratios,2));
            else
                tempPercentChange = tempRatios - Ratios(end,:);
            end;
            Ratios = [Ratios; tempRatios];
            PercentChange = [PercentChange; tempPercentChange];
            
            % Price * contract size = Price per contract
            tempSPHighPrice = tempOptionPriceSPHigh(:,2)*ContractSizeSPHigh;
            tempSPLowPrice = tempOptionPriceSPLow(:,2)*ContractSizeSPLow;
            SPContractHighPrice = [SPContractHighPrice; tempSPHighPrice];
            SPContractLowPrice = [SPContractLowPrice; tempSPLowPrice];
            
            % Put spread price
            ShortPutSpreadPrice = (tempOptionPriceSPLow(:,2)*ContractSizeSPLow - tempOptionPriceSPHigh(:,2)*ContractSizeSPHigh);
            VIXCallPrice = tempOptionPriceVIX(:,2)*ContractSizeVIX;
            
            % if rebalancing, calculate the number of VIX options
            % needed to cover price of put spread
            if date_index == rebal_date_serial(j)
                NumVIXOptions = -ShortPutSpreadPrice/VIXCallPrice;
            end
            
            % vix contract price = vix option price * number of options
            tempVIXContractPrice = VIXCallPrice*NumVIXOptions;
            VIXContractPrice = [VIXContractPrice; tempVIXContractPrice];
            tempDataStore = [tempSPHighName tempOptionPriceSPHigh(:,2) tempSPLowName tempOptionPriceSPLow(:,2) tempVIXName tempOptionPriceVIX(:,2) NumVIXOptions];
            DataStore = [DataStore; tempDataStore];
        end
        
        
    end
    
    SPContractHigh.(PortNum{p}) = SPContractHighPrice;
    SPContractLow.(PortNum{p}) = SPContractLowPrice;
    VIXContract.(PortNum{p}) = VIXContractPrice;
    PercChange.(PortNum{p}) = PercentChange;
    Rat.(PortNum{p}) = Ratios;
    PortDataStore.(PortNum{p}) = DataStore;
    %% calculate portfolio return
    for k = 1:size(DataStore,1)-1
        PortReturn(k,:) = (PercentChange(k+1,2)*SPContractHighPrice(k+1)*(-1) + PercentChange(k+1,3)*SPContractLowPrice(k+1)+ PercentChange(k+1,4)*VIXContractPrice(k+1))/StartValue;
    end
    PortValue.(PortNum{p}) = [Ratios(3:end,1) PortReturn]; %DataStore(2:end,:)];
    
end
toc

%find index where all 3 portfolios start
z = PortValue.(PortNum{end})(1,1);
DateStart = [];
for k = 1:length(PortNum)-1
    for i = 1:length(PortValue.(PortNum{k}))
        if (PortValue.(PortNum{k})(i,1)) == (z)
            DateStart(k) = i;
        end
    end
end

%calculate total portfolio return
ReturnIndex = 100;
for i = 1:length(PortValue.(PortNum{end}))
    TotalPortValue(i,:) = (PortValue.(PortNum{1})(DateStart(1)+i-1,2)) + (PortValue.(PortNum{2})(DateStart(2)+i-1,2)) + (PortValue.(PortNum{3})(i,2));
    ReturnIndex(i+1,:) = ReturnIndex(i)*(TotalPortValue(i)+1);
end


%plot total portfolio cumulative return
Dates = (PortValue.(PortNum{3})(:,1));
plot(Dates,ReturnIndex(2:end,:));
hold on
datetick('x','keepticks','keeplimits')
format bank
temptitle = ['Total Portfolio Value: ' {SPHighDelta} {SPLowDelta} {VIXDelta}];
title(temptitle)

end



