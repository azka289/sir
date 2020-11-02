package fmp.mapping;

import java.util.HashMap;
import java.util.LinkedList;

import fmp.contextEvolModel.AdaptationRuleWithCtxProposition;
import fmp.contextEvolModel.CKS;
import fmp.contextEvolModel.ContextProposition;
import fmp.contextModel.AdaptationRuleWithCtxFeature;
import fmp.featureModel.DSPL;
import fmp.featureModel.ExcludeRule;
import fmp.featureModel.Feature;
import fmp.featureModel.FeatureGroup;
import fmp.featureModel.FeatureGroupType;
import fmp.featureModel.FeatureType;
import fmp.featureModel.RequireRule;

public class LTLProperties {

	private String newline = System.getProperty("line.separator");
	
	private LinkedList<Feature> contextAwareFeatures;
	
	private DSPL dspl;
	
	private CKS cks;
	
	private HashMap<String,String> propertiesList; 
	
	public LTLProperties(LinkedList<Feature> contextAwareFeatures, DSPL dspl, CKS cks, HashMap<String,String> propertiesList){
		this.dspl = dspl;
		this.contextAwareFeatures = contextAwareFeatures;
		this.cks = cks;
		this.propertiesList = propertiesList; // it contains the string related to the properties that will be executed in Spin
	}
	
	public String getLTLPropertiestoPromelaCode() {
		StringBuffer ltlPropertiesPromelaCode = new StringBuffer();
		ltlPropertiesPromelaCode.append(appendLTL_configurationCorrectness());
		ltlPropertiesPromelaCode.append(appendLTL_RuleLiveness());
		ltlPropertiesPromelaCode.append(appendLTL_InterleavingCorrectness());
		ltlPropertiesPromelaCode.append(appendLTL_FeatureLiveness());
		ltlPropertiesPromelaCode.append(appendLTL_VariationLiveness());
		return ltlPropertiesPromelaCode.toString();
	}
	
	// Example: ltl pro01 {[] ((showDocs -> mobileGuide) && (text -> showDocs) &&  (image -> showDocs) && (video -> showDocs) && 
	// (mobileGuide -> showDocs) && (showDocs -> text) && (showDocs -> (image || video)) )}
	public String appendLTL_configurationCorrectness(){
		StringBuffer ltlConfigurationCorrectnessPromelaCode = new StringBuffer();
		ltlConfigurationCorrectnessPromelaCode.append("ltl pro1 {[] (");
		
		
		//rules son -> father
		for(int i = 0; i< dspl.getFeatures().size(); i++){
			Feature feature = dspl.getFeatures().get(i);
			if(feature.getFatherFeature()!= null){
				ltlConfigurationCorrectnessPromelaCode.append("("+feature.getName()+" -> "+feature.getFatherFeature().getName()+")");
				if(i < dspl.getFeatures().size()-2)
					ltlConfigurationCorrectnessPromelaCode.append(" && ");
			}
		}
		
		//mandatory
		//TODO		
		
		//or-group
		//xor-group
		for (FeatureGroup featureGroup : dspl.getFeatureGroups()) {
			ltlConfigurationCorrectnessPromelaCode.append(" && ("+featureGroup.getFeatures().getFirst().getFatherFeature().getName()
					+ " -> (");
			if(featureGroup.getGroupType().equals(FeatureGroupType.OR_Group)){
				for (Feature feature : featureGroup.getFeatures()) {
					ltlConfigurationCorrectnessPromelaCode.append(feature.getName()+" || ");
				}				
			}
			//bitwise exclusive or
			if(featureGroup.getGroupType().equals(FeatureGroupType.Alternative_Group)){
				for (Feature feature : featureGroup.getFeatures()) {
					ltlConfigurationCorrectnessPromelaCode.append(feature.getName()+" ^ ");
				}
			}
			// it replaces the last ('||' or '^') for '))'
			ltlConfigurationCorrectnessPromelaCode.replace(ltlConfigurationCorrectnessPromelaCode.length()-3,
					ltlConfigurationCorrectnessPromelaCode.length(),"))");
		}
		
		//exclude rule
		//(text -> !A && !B)
		for (ExcludeRule excludeRule : dspl.getExcludeRules()) {
			ltlConfigurationCorrectnessPromelaCode.append(" && ("+excludeRule.getFeatureClaimant().getName() +" -> (");
			for(int i = 0; i < excludeRule.getRequiredFeatureList().size(); i++){
				ltlConfigurationCorrectnessPromelaCode.append("!"+excludeRule.getRequiredFeatureList().get(i).getName());
				if(i < excludeRule.getRequiredFeatureList().size()-1)
					ltlConfigurationCorrectnessPromelaCode.append(" && ");
			}		
			ltlConfigurationCorrectnessPromelaCode.append("))");
		}
		
		//require rule
		for (RequireRule requireRule : dspl.getRequireRules()) {
			ltlConfigurationCorrectnessPromelaCode.append(" && ("+requireRule.getFeatureClaimant().getName() +" -> (");
			for(int i = 0; i < requireRule.getRequiredFeatureList().size(); i++){
				ltlConfigurationCorrectnessPromelaCode.append(requireRule.getRequiredFeatureList().get(i).getName());
				if(i < requireRule.getRequiredFeatureList().size()-1)
					ltlConfigurationCorrectnessPromelaCode.append(" && ");
			}		
			ltlConfigurationCorrectnessPromelaCode.append("))");
		}
	
		ltlConfigurationCorrectnessPromelaCode.append(")}"+newline);
		
		propertiesList.put("pro1",ltlConfigurationCorrectnessPromelaCode.toString());
		return ltlConfigurationCorrectnessPromelaCode.toString();
	}
	
	//Example: ltl pro21 {<> (battery == 1 && hasPwSrc == false)}
	public String appendLTL_RuleLiveness(){
		StringBuffer ltlRuleLivenessPromelaCode = new StringBuffer();
		int i = 1;
		for (AdaptationRuleWithCtxProposition adaptationRule : cks.getAdaptationRules()) {
			ltlRuleLivenessPromelaCode.append("ltl pro2"+i+" {<> (");
			for (ContextProposition contextProposition : adaptationRule.getContextRequired()) {
				ltlRuleLivenessPromelaCode.append(contextProposition.getName()+" == true && ");
			}
			
			//it replaces the last '&&' for ')}'
			ltlRuleLivenessPromelaCode.replace(ltlRuleLivenessPromelaCode.length()-3,ltlRuleLivenessPromelaCode.length(),")}"+newline);
			
			String ltlRuleLivProp[] = ltlRuleLivenessPromelaCode.toString().split(newline); 
			propertiesList.put("pro2"+i,ltlRuleLivProp[i-1]);
			
			i++;
			
		}
		return ltlRuleLivenessPromelaCode.toString();
	}
	
	//ltl pro31 {[] ((battery == 1 && hasPwSrc == false) -> <>(image == false && video == false))}
	public String appendLTL_InterleavingCorrectness(){
		StringBuffer ltlInterleavingCorrectnessPromelaCode = new StringBuffer();
		int i = 1;
		for (AdaptationRuleWithCtxProposition adaptationRule : cks.getAdaptationRules()) {
			
			ltlInterleavingCorrectnessPromelaCode.append("ltl pro3"+i+" {[] ((");
			// Context Trigger
			for (ContextProposition contextProposition : adaptationRule.getContextRequired()) {
				ltlInterleavingCorrectnessPromelaCode.append(contextProposition.getName()+" == true && ");
			}
			// it replaces the last '&&' for ') -> <> ('
			ltlInterleavingCorrectnessPromelaCode.replace(ltlInterleavingCorrectnessPromelaCode.length()-3,ltlInterleavingCorrectnessPromelaCode.length(),") -> <> (");
			// Features to active
			for (Feature toActiveFeature : adaptationRule.getToActiveFeatureList()) {
				ltlInterleavingCorrectnessPromelaCode.append(toActiveFeature.getName()+" == true && ");
			}
			
			// Features to deactive
			for (Feature toDeactiveFeature : adaptationRule.getToDeactiveFeatureList()) {
				ltlInterleavingCorrectnessPromelaCode.append(toDeactiveFeature.getName()+" == false && ");
			}		
			
			// it replaces the last '&&' for '))}'
			ltlInterleavingCorrectnessPromelaCode.replace(ltlInterleavingCorrectnessPromelaCode.length()-4,
					ltlInterleavingCorrectnessPromelaCode.length(),"))}"+newline);
			
			String ltlRuleLivProp[]= ltlInterleavingCorrectnessPromelaCode.toString().split(newline); 
			propertiesList.put("pro3"+i,ltlRuleLivProp[i-1]);
			
			i++;
			
		}
		return ltlInterleavingCorrectnessPromelaCode.toString();
	}
	
	//ltl pro42 {<> video}
	public String appendLTL_FeatureLiveness(){
		StringBuffer ltlFeatureLivenessPromelaCode = new StringBuffer();
		int i = 1;
		for (Feature feature : contextAwareFeatures) {
			ltlFeatureLivenessPromelaCode.append("ltl pro4"+i+" {<> "+feature.getName()+"}"+newline);
			
			String ltlRuleLivProp[]= ltlFeatureLivenessPromelaCode.toString().split(newline); 
			propertiesList.put("pro4"+i,ltlRuleLivProp[i-1]);
			
			i++;
		}
		return ltlFeatureLivenessPromelaCode.toString();
	}
	
	//Example: ltl pro52 {<> !video}
	public String appendLTL_VariationLiveness(){
		StringBuffer ltlVariationLivenessPromelaCode = new StringBuffer();
		int i = 1;
		for (Feature feature : contextAwareFeatures) {
			ltlVariationLivenessPromelaCode.append("ltl pro5"+i+" {<> !"+feature.getName()+"}"+newline);
			
			String ltlRuleLivProp[]= ltlVariationLivenessPromelaCode.toString().split(newline); 
			propertiesList.put("pro5"+i,ltlRuleLivProp[i-1]);
			
			i++;
		}
		return ltlVariationLivenessPromelaCode.toString();
	}
	
}
