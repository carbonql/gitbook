# Summary

## Overview

* [Carbon QL Cookbook](README.md)
  * [How does it work?](README.md#how-does-it-work)
  * [Experimental syntax extensions for JavaScript](README.md#experimental-js)
  * [Structure of this gitbook](README.md#structure-of-book)

## Cookbook
* [**1** Query examples](babel/README.md)
  * [**1.1** Governance Queries](babel/governance-queries.md)
    * [Audit all Certificates, including status, user, and requested usages](babel/governance-queries.md#certsignrequests)
    * [Distinct versions of `mysql` container in cluster](babel/governance-queries.md#distinctmysqlversions)
    * [List all Namespaces with no hard memory quota specified](babel/governance-queries.md#namespaceswithnomemquota)
    * [Pods using the default ServiceAccount](babel/governance-queries.md#podsusingdefaultserviceaccount)
    * [Find Services publicly exposed to the Internet](babel/governance-queries.md#servicespubliclyexposed)
    * [Find users and ServiceAccounts with access to Secrets](babel/governance-queries.md#userswithsecretaccess)
    * [Aggregate high-level report on resource consumption in a Namespace](babel/governance-queries.md#aggregatereportonnamespace)
    * [List Pods and their ServiceAccount \(possibly a unique user\) by Secrets they use](babel/governance-queries.md#podsandsvcacctbysecretuse)
    * [List Pods grouped by PersistentVolumes they use](babel/governance-queries.md#podsbypvuse)
  * [**1.2** Operations Queries](babel/operations.md)
    * [Find all Pod logs containing](babel/operations.md#errorlogsgroupedbypod)
    * [Diff last two rollouts of an application](babel/operations.md#historyofdeployment)
    * [Find all Pods scheduled on nodes with high memory pressure](babel/operations.md#podsonnodeswithmempressure)
    * [Aggregate cluster-wide error and warning Events into a report](babel/operations.md#aggregatereportonnamespace)
